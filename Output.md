## \tsconfig.json
```ts
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "outDir": "build",
    "rootDir": "src",
    "strict": true,
    "moduleResolution": "Node",
    "allowSyntheticDefaultImports": true,

    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,

    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "exclude": ["build", "node_modules"]
}

```

## \src\main.ts
```ts
import "dotenv/config";
import mongoose from "mongoose";
import { createLogger, format, transports } from "winston";

import { Client } from "discordx";
import { dirname, importx } from "@discordx/importer";
import { NotBot } from "@discordx/utilities";

import type { Interaction } from "discord.js";
import {
  ActivityType,
  Colors,
  EmbedBuilder,
  IntentsBitField,
  Partials,
} from "discord.js";

import { replyOrFollowUp } from "./utils/others.js";

const { BOT_TOKEN, MONGO_URI } = process.env;

if (!BOT_TOKEN) {
  throw new Error("BOT_TOKEN is not set in the environment variables.");
}

if (!MONGO_URI) {
  throw new Error("MONGO_URI is not set in the environment variables.");
}

// Structured logging setup
const logger = createLogger({
  level: "info",
  format: format.json(),
  defaultMeta: { service: "user-service" },
  transports: [new transports.Console()],
});

export const bot = new Client({
  // To use only guild command
  // botGuilds: [(client) => client.guilds.cache.map((guild) => guild.id)],

  intents: [
    IntentsBitField.Flags.Guilds,
    IntentsBitField.Flags.GuildMembers,
    IntentsBitField.Flags.GuildMessages,
    IntentsBitField.Flags.GuildMessageReactions,
    IntentsBitField.Flags.GuildVoiceStates,
    IntentsBitField.Flags.MessageContent,
  ],
  partials: [Partials.Channel, Partials.GuildMember, Partials.User],

  silent: false,
  guards: [NotBot],
  presence: {
    activities: [
      {
        name: `Dev Mode`,
        type: ActivityType.Watching,
      },
    ],
    status: "online",
  },
});

bot.once("ready", async () => {
  try {
    await bot.guilds.fetch();
    await bot.clearApplicationCommands();
    await bot.initApplicationCommands();
    logger.info(`Logged in as ${bot.user!.tag}`);
  } catch (error) {
    logger.error("An error occurred: ", error);
  }
});

bot.on("interactionCreate", handleInteraction);

async function handleInteraction(interaction: Interaction) {
  if (interaction.isButton()) {
    if (interaction.customId.endsWith("cancel_move"))
      return replyOrFollowUp(interaction, {
        embeds: [
          EmbedBuilder.from(interaction.message.embeds[0])
            .setColor(Colors.Red)
            .setTitle("Cancelled")
            .setDescription(`Action cancelled.`),
        ],
        components: [],
        ephemeral: true,
      });
    if (interaction.customId.includes("pagination")) return;
  }

  bot.executeInteraction(interaction);
}

try {
  const mongoClient = await mongoose.connect(MONGO_URI);
  logger.info(
    `Disconnected from previous MongoDB connection and now connected to ${mongoClient.connection.db.databaseName} database`
  );
} catch (error) {
  logger.error("An error occurred when connecting to MongoDB: ", error);
}

await importx(`${dirname(import.meta.url)}/{events,commands}/**/*.{ts,js}`);
await bot.login(BOT_TOKEN);
```

## \src\commands\admin\blacklist.ts
```ts
import {
  Discord,
  Slash,
  SlashGroup,
  SlashOption,
  ButtonComponent,
  Guard,
  ContextMenu,
} from "discordx";
import { Category, RateLimit, TIME_UNIT } from "@discordx/utilities";

import {
  CommandInteraction,
  ApplicationCommandOptionType,
  User,
  Role,
  ButtonInteraction,
  GuildBasedChannel,
  ApplicationCommandType,
  UserContextMenuCommandInteraction,
  MessageContextMenuCommandInteraction,
} from "discord.js";

import { ServerModel } from "../../models/ServerModel.js";
import { moderationHierachy } from "../../utils/moderationHierachy.js";
import {
  ButtonComponentMoveSnowflake,
  PaginationSender,
} from "../../utils/PaginationButtons.js";

@Discord()
@Category("Admin Commands")
@SlashGroup({
  description: "Manage blacklist for the guild or in a specific command",
  name: "blacklist",
})
@SlashGroup("blacklist")
class Blacklist {
  @Slash({ description: "View the blacklist", name: `view` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async view(
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    PaginationSender({
      server,
      list: "blacklist",
      snowflakePluralType: "guilds",
      interaction,
      commandName,
    });
  }
}

@Discord()
@SlashGroup({
  description: "User blacklist",
  name: "user",
  root: "blacklist",
})
@SlashGroup("user", "blacklist")
class UserBlacklist {
  @ButtonComponent({ id: "blacklist_user_move_target" })
  async moveUser(interaction: ButtonInteraction) {
    console.log(
      "🚀 ~ file: blacklist.ts:95 ~ UserBlacklist ~ moveUser ~ moveUser:",
      Date.now()
    );
    ButtonComponentMoveSnowflake(interaction);
  }
  @Slash({ description: "Add a user to the blacklist", name: `add` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async add(
    @SlashOption({
      name: `target`,
      type: ApplicationCommandOptionType.User,
      required: true,
      description: "Add a user to the blacklist",
    })
    target: User,
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const fairCheck = await moderationHierachy(target, interaction).catch((e) =>
      console.log(e)
    );
    if (fairCheck)
      return interaction.reply({ content: fairCheck, ephemeral: true });

    server.cases.blacklist.applicationModifySelection({
      type: target,
      commandName,
      interaction,
      list: "blacklist",
      action: "add",
    });
  }

  @ContextMenu({
    name: `Add to blacklist`,
    type: ApplicationCommandType.User,
  })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async contextUserAdd(interaction: UserContextMenuCommandInteraction) {
    const target = interaction.targetUser;

    await interaction.deferReply({ ephemeral: true });

    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const fairCheck = await moderationHierachy(target, interaction).catch((e) =>
      console.log(e)
    );
    if (fairCheck) return interaction.editReply({ content: fairCheck });

    server.cases.blacklist.applicationModifySelection({
      type: target,
      interaction,
      list: "blacklist",
      action: "add",
    });
  }

  @ContextMenu({
    name: `Add to blacklist`,
    type: ApplicationCommandType.Message,
  })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async contextMessageAdd(interaction: MessageContextMenuCommandInteraction) {
    const target = interaction.targetMessage.author;

    await interaction.deferReply({ ephemeral: true });

    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const fairCheck = await moderationHierachy(target, interaction).catch((e) =>
      console.log(e)
    );
    if (fairCheck) return interaction.editReply({ content: fairCheck });

    server.cases.blacklist.applicationModifySelection({
      type: target,
      interaction,
      list: "blacklist",
      action: "add",
    });
  }

  @Slash({ description: "Remove a user from the blacklist", name: `remove` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async remove(
    @SlashOption({
      name: `target`,
      type: ApplicationCommandOptionType.User,
      description: `Remove a user from the blacklist`,
      required: true,
    })
    target: User,
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const fairCheck = await moderationHierachy(target, interaction).catch((e) =>
      console.log(e)
    );
    if (fairCheck)
      return interaction.reply({ content: fairCheck, ephemeral: true });

    server.cases.blacklist.applicationModifySelection({
      type: target,
      commandName,
      interaction,
      list: "blacklist",
      action: "remove",
    });
  }

  @ContextMenu({
    name: `Remove from blacklist`,
    type: ApplicationCommandType.User,
  })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async contextUserRemove(interaction: UserContextMenuCommandInteraction) {
    const target = interaction.targetUser;

    await interaction.deferReply({ ephemeral: true });

    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const fairCheck = await moderationHierachy(target, interaction).catch((e) =>
      console.log(e)
    );
    if (fairCheck) return interaction.editReply({ content: fairCheck });

    server.cases.blacklist.applicationModifySelection({
      type: target,
      interaction,
      list: "blacklist",
      action: "remove",
    });
  }

  @ContextMenu({
    name: `Remove from blacklist`,
    type: ApplicationCommandType.Message,
  })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async contextMessageRemove(
    interaction: MessageContextMenuCommandInteraction
  ) {
    const target = interaction.targetMessage.author;

    await interaction.deferReply({ ephemeral: true });

    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const fairCheck = await moderationHierachy(target, interaction).catch((e) =>
      console.log(e)
    );
    if (fairCheck) return interaction.editReply({ content: fairCheck });

    server.cases.blacklist.applicationModifySelection({
      type: target,
      interaction,
      list: "blacklist",
      action: "remove",
    });
  }

  @Slash({ description: "View the blacklist", name: `view` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async view(
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    PaginationSender({
      server,
      list: "blacklist",
      snowflakePluralType: "users",
      interaction,
      commandName,
    });
  }
}

@Discord()
@SlashGroup({
  description: "Role blacklist",
  name: "role",
  root: "blacklist",
})
@SlashGroup("role", "blacklist")
class RoleBlacklist {
  @ButtonComponent({ id: "blacklist_role_move_target" })
  async moveUser(interaction: ButtonInteraction) {
    console.log(
      "🚀 ~ file: blacklist.ts:255 ~ RoleBlacklist ~ moveUser ~ moveUser:",
      Date.now()
    );
    ButtonComponentMoveSnowflake(interaction);
  }
  @Slash({ description: "Add a role to the blacklist", name: `add` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async add(
    @SlashOption({
      name: `target`,
      type: ApplicationCommandOptionType.Role,
      required: true,
      description: "Add a role to the blacklist",
    })
    target: Role,
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const fairCheck = await moderationHierachy(target, interaction).catch((e) =>
      console.log(e)
    );
    if (fairCheck)
      return interaction.reply({ content: fairCheck, ephemeral: true });

    server.cases.blacklist.applicationModifySelection({
      type: target,
      commandName,
      interaction,
      list: "blacklist",
      action: "add",
    });
  }

  @Slash({ description: "Remove a role from the blacklist", name: `remove` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async remove(
    @SlashOption({
      name: `target`,
      type: ApplicationCommandOptionType.Role,
      description: `Remove a role from the blacklist`,
      required: true,
    })
    target: Role,
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const fairCheck = await moderationHierachy(target, interaction).catch((e) =>
      console.log(e)
    );
    if (fairCheck)
      return interaction.reply({ content: fairCheck, ephemeral: true });

    server.cases.blacklist.applicationModifySelection({
      type: target,
      commandName,
      interaction,
      list: "blacklist",
      action: "remove",
    });
  }

  @Slash({ description: "View the blacklist", name: `view` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async view(
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    PaginationSender({
      server,
      list: "blacklist",
      snowflakePluralType: "roles",
      interaction,
      commandName,
    });
  }
}

@Discord()
@SlashGroup({
  description: "Channel blacklist",
  name: "channel",
  root: "blacklist",
})
@SlashGroup("channel", "blacklist")
class ChannelBlacklist {
  @ButtonComponent({ id: "blacklist_channel_move_target" })
  async moveUser(interaction: ButtonInteraction) {
    console.log(
      "🚀 ~ file: blacklist.ts:410 ~ ChannelBlacklist ~ moveUser ~ interaction:",
      Date.now()
    );

    ButtonComponentMoveSnowflake(interaction);
  }
  @Slash({ description: "Add a channel to the blacklist", name: `add` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async add(
    @SlashOption({
      name: `target`,
      type: ApplicationCommandOptionType.Channel,
      required: true,
      description: "Add a channel to the blacklist",
    })
    target: GuildBasedChannel,
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    server.cases.blacklist.applicationModifySelection({
      type: target,
      commandName,
      interaction,
      list: "blacklist",
      action: "add",
    });
  }

  @Slash({ description: "Remove a channel from the blacklist", name: `remove` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async remove(
    @SlashOption({
      name: `target`,
      type: ApplicationCommandOptionType.Channel,
      description: `Remove a channel from the blacklist`,
      required: true,
    })
    target: GuildBasedChannel,
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    server.cases.blacklist.applicationModifySelection({
      type: target,
      commandName,
      interaction,
      list: "blacklist",
      action: "remove",
    });
  }

  @Slash({ description: "View the blacklist", name: `view` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async view(
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    PaginationSender({
      server,
      list: "blacklist",
      snowflakePluralType: "channels",
      interaction,
      commandName,
    });
  }
}
```

## \src\commands\admin\whitelist.ts
```ts
import {
  Discord,
  Slash,
  SlashGroup,
  SlashOption,
  ButtonComponent,
  Guard,
  ContextMenu,
} from "discordx";
import { Category, RateLimit, TIME_UNIT } from "@discordx/utilities";
import {
  CommandInteraction,
  ApplicationCommandOptionType,
  User,
  Role,
  ButtonInteraction,
  GuildBasedChannel,
  UserContextMenuCommandInteraction,
  ApplicationCommandType,
  MessageContextMenuCommandInteraction,
} from "discord.js";

import { ServerModel } from "../../models/ServerModel.js";
import { moderationHierachy } from "../../utils/moderationHierachy.js";
import {
  ButtonComponentMoveSnowflake,
  PaginationSender,
} from "../../utils/PaginationButtons.js";

@Discord()
@Category("Admin Commands")
@SlashGroup({
  description: "Manage whitelist for the guild or in a specific command",
  name: "whitelist",
})
@SlashGroup("whitelist")
class Whitelist {
  @Slash({ description: "View the whitelist", name: `view` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async view(
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    PaginationSender({
      server,
      list: "whitelist",
      snowflakePluralType: "guilds",
      interaction,
      commandName,
    });
  }
}

@Discord()
@SlashGroup({
  description: "User whitelist",
  name: "user",
  root: "whitelist",
})
@SlashGroup("user", "whitelist")
class UserWhitelist {
  @ButtonComponent({ id: "whitelist_user_move_target" })
  async moveUser(interaction: ButtonInteraction) {
    console.log(
      "🚀 ~ file: whitelist.ts:95 ~ UserWhitelist ~ moveUser ~ moveUser:",
      Date.now()
    );
    ButtonComponentMoveSnowflake(interaction);
  }
  @Slash({ description: "Add a user to the whitelist", name: `add` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async add(
    @SlashOption({
      name: `target`,
      type: ApplicationCommandOptionType.User,
      required: true,
      description: "Add a user to the whitelist",
    })
    target: User,
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const fairCheck = await moderationHierachy(target, interaction)
      .catch((e) => console.log(e))
      .catch((e) => console.log(e));
    if (fairCheck)
      return interaction.reply({ content: fairCheck, ephemeral: true });

    server.cases.whitelist.applicationModifySelection({
      type: target,
      commandName,
      interaction,
      list: "whitelist",
      action: "add",
    });
  }

  @ContextMenu({
    name: `Add to whitelist`,
    type: ApplicationCommandType.User,
  })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async contextUserAdd(interaction: UserContextMenuCommandInteraction) {
    const target = interaction.targetUser;

    await interaction.deferReply({ ephemeral: true });
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const fairCheck = await moderationHierachy(target, interaction)
      .catch((e) => console.log(e))
      .catch((e) => console.log(e));
    if (fairCheck) return interaction.editReply({ content: fairCheck });

    server.cases.whitelist.applicationModifySelection({
      type: target,
      interaction,
      list: "whitelist",
      action: "add",
    });
  }

  @ContextMenu({
    name: `Add to whitelist`,
    type: ApplicationCommandType.Message,
  })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async contextMessageAdd(interaction: MessageContextMenuCommandInteraction) {
    const target = interaction.targetMessage.author;

    await interaction.deferReply({ ephemeral: true });

    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const fairCheck = await moderationHierachy(target, interaction)
      .catch((e) => console.log(e))
      .catch((e) => console.log(e));
    if (fairCheck) return interaction.editReply({ content: fairCheck });

    server.cases.whitelist.applicationModifySelection({
      type: target,
      interaction,
      list: "whitelist",
      action: "add",
    });
  }

  @Slash({ description: "Remove a user from the whitelist", name: `remove` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async remove(
    @SlashOption({
      name: `target`,
      type: ApplicationCommandOptionType.User,
      description: `Remove a user from the whitelist`,
      required: true,
    })
    target: User,
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const fairCheck = await moderationHierachy(target, interaction)
      .catch((e) => console.log(e))
      .catch((e) => console.log(e));
    if (fairCheck)
      return interaction.reply({ content: fairCheck, ephemeral: true });

    server.cases.whitelist.applicationModifySelection({
      type: target,
      commandName,
      interaction,
      list: "whitelist",
      action: "remove",
    });
  }

  @ContextMenu({
    name: `Remove from whitelist`,
    type: ApplicationCommandType.User,
  })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async contextRemove(interaction: UserContextMenuCommandInteraction) {
    const target = interaction.targetUser;

    await interaction.deferReply({ ephemeral: true });

    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const fairCheck = await moderationHierachy(target, interaction).catch((e) =>
      console.log(e)
    );
    if (fairCheck) return interaction.editReply({ content: fairCheck });

    server.cases.whitelist.applicationModifySelection({
      type: target,
      interaction,
      list: "whitelist",
      action: "remove",
    });
  }

  @ContextMenu({
    name: `Remove from whitelist`,
    type: ApplicationCommandType.Message,
  })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async contextMessageRemove(
    interaction: MessageContextMenuCommandInteraction
  ) {
    const target = interaction.targetMessage.author;

    await interaction.deferReply({ ephemeral: true });

    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const fairCheck = await moderationHierachy(target, interaction).catch((e) =>
      console.log(e)
    );
    if (fairCheck)
      return interaction.reply({ content: fairCheck, ephemeral: true });

    server.cases.whitelist.applicationModifySelection({
      type: target,
      interaction,
      list: "whitelist",
      action: "remove",
    });
  }

  @Slash({ description: "View the whitelist", name: `view` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async view(
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    PaginationSender({
      server,
      list: "whitelist",
      snowflakePluralType: "users",
      interaction,
      commandName,
    });
  }
}

@Discord()
@SlashGroup({
  description: "Role whitelist",
  name: "role",
  root: "whitelist",
})
@SlashGroup("role", "whitelist")
class RoleWhitelist {
  @ButtonComponent({ id: "whitelist_role_move_target" })
  async moveUser(interaction: ButtonInteraction) {
    console.log(
      "🚀 ~ file: whitelist.ts:254 ~ RoleWhitelist ~ moveUser ~ moveUser:",
      Date.now()
    );
    ButtonComponentMoveSnowflake(interaction);
  }
  @Slash({ description: "Add a role to the whitelist", name: `add` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async add(
    @SlashOption({
      name: `target`,
      type: ApplicationCommandOptionType.Role,
      required: true,
      description: "Add a role to the whitelist",
    })
    target: Role,
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const fairCheck = await moderationHierachy(target, interaction).catch((e) =>
      console.log(e)
    );
    if (fairCheck)
      return interaction.reply({ content: fairCheck, ephemeral: true });

    server.cases.whitelist.applicationModifySelection({
      type: target,
      commandName,
      interaction,
      list: "whitelist",
      action: "add",
    });
  }

  @Slash({ description: "Remove a role from the whitelist", name: `remove` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async remove(
    @SlashOption({
      name: `target`,
      type: ApplicationCommandOptionType.Role,
      description: `Remove a role from the whitelist`,
      required: true,
    })
    target: Role,
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const fairCheck = await moderationHierachy(target, interaction).catch((e) =>
      console.log(e)
    );
    if (fairCheck)
      return interaction.reply({ content: fairCheck, ephemeral: true });

    server.cases.whitelist.applicationModifySelection({
      type: target,
      commandName,
      interaction,
      list: "whitelist",
      action: "remove",
    });
  }

  @Slash({ description: "View the whitelist", name: `view` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async view(
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    PaginationSender({
      server,
      list: "whitelist",
      snowflakePluralType: "roles",
      interaction,
      commandName,
    });
  }
}

@Discord()
@SlashGroup({
  description: "Channel whitelist",
  name: "channel",
  root: "whitelist",
})
@SlashGroup("channel", "whitelist")
class ChannelWhitelist {
  @ButtonComponent({ id: "whitelist_channel_move_target" })
  async moveUser(interaction: ButtonInteraction) {
    console.log(
      "🚀 ~ file: whitelist.ts:411 ~ ChannelWhitelist ~ moveUser ~ interaction:",
      Date.now()
    );
    ButtonComponentMoveSnowflake(interaction);
  }
  @Slash({ description: "Add a channel to the whitelist", name: `add` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async add(
    @SlashOption({
      name: `target`,
      type: ApplicationCommandOptionType.Channel,
      required: true,
      description: "Add a channel to the whitelist",
    })
    target: GuildBasedChannel,
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    server.cases.whitelist.applicationModifySelection({
      type: target,
      commandName,
      interaction,
      list: "whitelist",
      action: "add",
    });
  }

  @Slash({ description: "Remove a channel from the whitelist", name: `remove` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async remove(
    @SlashOption({
      name: `target`,
      type: ApplicationCommandOptionType.Channel,
      description: `Remove a channel from the whitelist`,
      required: true,
    })
    target: GuildBasedChannel,
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    server.cases.whitelist.applicationModifySelection({
      type: target,
      commandName,
      interaction,
      list: "whitelist",
      action: "remove",
    });
  }

  @Slash({ description: "View the whitelist", name: `view` })
  @Guard(RateLimit(TIME_UNIT.seconds, 3, { ephemeral: true }))
  async view(
    @SlashOption({
      description: `The command name`,
      name: "command",
      type: ApplicationCommandOptionType.String,
    })
    commandName: string | undefined,
    interaction: CommandInteraction
  ) {
    let server = await ServerModel.findOne({
      serverId: interaction.guildId!,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    PaginationSender({
      server,
      list: "whitelist",
      snowflakePluralType: "channels",
      interaction,
      commandName,
    });
  }
}
```

## \src\events\common.ts
```ts
import type { ArgsOf, Client } from "discordx";
import { Discord, On } from "discordx";

@Discord()
export class Example {
  @On()
  messageDelete([message]: ArgsOf<"messageDelete">, client: Client): void {
    console.log("Message Deleted", client.user?.username, message.content);
  }
}
```

## \src\models\ServerModel.ts
```ts
import {
  DocumentType,
  SubDocumentType,
  getModelForClass,
  modelOptions,
  post,
  pre,
  prop,
} from "@typegoose/typegoose";
import { TimeStamps } from "@typegoose/typegoose/lib/defaultClasses.js";
import { BeAnObject } from "@typegoose/typegoose/lib/types.js";
import { ChangeStreamDocument } from "mongodb";
import { Document } from "mongoose";
import { Blacklist } from "./Moderation/AccessGate/Blacklist.js";
import { Whitelist } from "./Moderation/AccessGate/Whitelist.js";
import { ModerationCases } from "./Moderation/ModerationCases.js";

@modelOptions({
  schemaOptions: {
    timestamps: true,
  },
})
export class SnowflakeLog {
  @prop({ required: true })
  public readonly id!: string;

  @prop()
  public expired?: boolean;
}

/**
 * User class
 * Represents a user in the system
 */
@pre<User>("save", function () {
  // This pre-save hook will run before a document is saved
  if (this.name.endsWith("#0")) this.name = this.name.slice(0, -2);
})
export class User extends SnowflakeLog {
  /** Discord's new username system means username is from
   * ```ts
   * <User>.tag
   * ```
   */
  @prop({ required: true })
  public name!: string;
}

export class Role extends SnowflakeLog {
  @prop({ required: true })
  public name!: string;
}

export class Channel extends Role {}

@pre<Server>("save", function (next) {
  // This pre-save hook will run before a document is saved
  console.log("A server document is going to be saved.");
  next();
})
@post<Server>("save", function (doc: DocumentType<Server>) {
  // This post-save hook will run after a document is saved
  console.log("A server document has been saved.", doc.toJSON());
})
export class Server extends TimeStamps {
  @prop({ required: true })
  public readonly serverId!: string;

  @prop({ type: () => ModerationCases, default: {} })
  public cases!: SubDocumentType<ModerationCases>;

  @prop({ required: true })
  public serverName!: string;

  @prop({ type: () => User, required: true })
  public createdBy!: SubDocumentType<User>;
}

export const ServerModel = getModelForClass(Server);
export const UserModel = getModelForClass(User);
export const BlacklistModel = getModelForClass(Blacklist);
export const WhitelistModel = getModelForClass(Whitelist);

const ServerModelChangeStream =
  ServerModel.watch<Document<unknown, BeAnObject, Server>>();

ServerModelChangeStream.on("change", (change: ChangeStreamDocument<Server>) => {
  if (change.operationType === "insert" && change.fullDocument.cases.actions) {
    const action =
      change.fullDocument.cases.actions[
        change.fullDocument.cases.actions.length - 1
      ];
    console.log(`A new Action was added: ${action}`);

    // TODO: Log the action to the Discord server using the 'serverID' property
  }
});


```

## \src\models\typings.ts
```ts
import {
  User as UserObj,
  Role as RoleObj,
  Channel as ChannelObj,
} from "./ServerModel.js";
import {
  BeAnObject,
  IObjectWithTypegooseFunction,
} from "@typegoose/typegoose/lib/types";
import {
  APIMessageComponentEmoji,
  ButtonStyle,
  GuildBasedChannel,
  Role,
  User,
} from "discord.js";
import mongoose, { Types } from "mongoose";

export type ModelUpdate<I> = {
  [K in keyof Omit<I, "_id" | "__v">]?: I[K];
};

export type AccessGateSubGroupApplicationCommandOptionType =
  | User
  | Role
  | GuildBasedChannel;

export type ServerModelSelectionSnowflakeType = UserObj | RoleObj | ChannelObj;
export type TargetClass = `users` | `roles` | `channels`;
export type TargetType = `User` | `Channel` | `Role`;

export type MongooseDocType<T = any> = mongoose.Document<
  unknown,
  BeAnObject,
  T
> &
  Omit<
    T & {
      _id: Types.ObjectId;
    },
    "typegooseName"
  > &
  IObjectWithTypegooseFunction;

export type FilteredKeys<T> = {
  [K in keyof T as T[K] extends Function ? never : K]: T[K];
};

export type AccessListBarrier = `blacklist` | `whitelist`;
export type TargetClassSingular = `user` | `role` | `channel`;

type ButtonIDPrefix = `${AccessListBarrier}_${TargetClassSingular | `guild`}_`;

export type ButtonIDFormat<T extends string = string> = T extends string
  ? `${ButtonIDPrefix}${T}`
  : ButtonIDPrefix;

export interface ButtonOptions {
  /**
   * Button emoji
   */
  emoji?: APIMessageComponentEmoji;
  /**
   * Button id
   */
  id?: string;
  /**
   * Button label
   */
  label?: string;
  /**
   * Button style
   */
  style?: ButtonStyle;
}

export type ButtonPaginationPositions = {
  start: ButtonOptions;
  next: ButtonOptions;
  previous: ButtonOptions;
  end: ButtonOptions;
  exit: ButtonOptions;
};

export enum ActionType {
  MUTE = "mute",
  KICK = "kick",
  BAN = "ban",
}
```

## \src\models\Moderation\Action.ts
```ts
import { SubDocumentType, pre, prop, queryMethod } from "@typegoose/typegoose";
import { TimeStamps } from "@typegoose/typegoose/lib/defaultClasses";
import { ActionType } from "../typings.js";
import { User } from "../ServerModel.js";
import { CounterModel } from "./Counter.js";
import { AsQueryMethod, QueryHelperThis } from "@typegoose/typegoose/lib/types.js";

@pre<Action>("save", async function (next) {
  try {
    const counter = await CounterModel.findByIdAndUpdate(
      { _id: "caseNumber" },
      { $inc: { seq: 1 } },
      { new: true, upsert: true }
    );
    this.caseNumber = counter.seq;
    next();
  } catch (error: any) {
    return next(error);
  }
})
export class Action extends TimeStamps {
  @prop({ type: () => User, required: true })
  public target!: SubDocumentType<User>;

  @prop({ type: () => User, required: true })
  public executor!: SubDocumentType<User>;

  @prop({ enum: () => ActionType, required: true })
  public type!: ActionType;

  @prop()
  public reason?: string;

  @prop()
  public caseNumber?: number;
}
```

## \src\models\Moderation\Counter.ts
```ts
import { prop, getModelForClass } from "@typegoose/typegoose";

export class Counter {
  @prop({ default: 0 })
  public seq?: number;
}

export const CounterModel = getModelForClass(Counter);
```

## \src\models\Moderation\ModerationCases.ts
```ts
import {
  ArraySubDocumentType,
  DocumentType,
  PropType,
  SubDocumentType,
  prop,
} from "@typegoose/typegoose";
import { Whitelist } from "./AccessGate/Whitelist.js";
import { Blacklist } from "./AccessGate/Blacklist.js";
import { Action } from "./Action.js";
import { ReturnModelType } from "@typegoose/typegoose";

export class ModerationCases {
  @prop({ type: () => Whitelist, default: {} })
  public whitelist!: SubDocumentType<Whitelist>;

  @prop({ type: () => Blacklist, default: {} })
  public blacklist!: SubDocumentType<Blacklist>;

  @prop({ type: () => [Action], default: [] }, PropType.ARRAY)
  public actions!: ArraySubDocumentType<Action>[];

  public async addAction(
    this: DocumentType<ModerationCases>,
    actionProps: Action
  ) {
    this.actions.push(actionProps as ArraySubDocumentType<Action>);
    await this.save();
  }

  public async removeAction(
    this: DocumentType<ModerationCases>,
    caseNumber: number
  ) {
    this.actions = this.actions.filter(
      (action) => action.caseNumber != caseNumber
    );
    await this.save();
  }

  public static async findByCaseNumber(
    this: ReturnModelType<typeof ModerationCases>,
    caseNumber: number
  ) {
    return this.findOne({
      $or: [
        { "whitelist.caseNumber": caseNumber },
        { "blacklist.caseNumber": caseNumber },
        { actions: { $elemMatch: { caseNumber: caseNumber } } },
      ],
    });
  }
}
```

## \src\models\Moderation\AccessGate\AccessGate.ts
```ts
import { prop, ArraySubDocumentType, PropType } from "@typegoose/typegoose";
import { Channel, Role, User } from "../../ServerModel.js";
import { TimeStamps } from "@typegoose/typegoose/lib/defaultClasses.js";

export class AccessSelection extends TimeStamps {
  @prop({ type: () => [User], default: [] }, PropType.ARRAY)
  public users!: ArraySubDocumentType<User>[];

  @prop({ type: () => [Role], default: [] }, PropType.ARRAY)
  public roles!: ArraySubDocumentType<Role>[];

  @prop({ type: () => [Channel], default: [] }, PropType.ARRAY)
  public channels!: ArraySubDocumentType<Channel>[];
}
```

## \src\models\Moderation\AccessGate\Blacklist.ts
```ts
import {
  pre,
  post,
  prop,
  DocumentType,
  ArraySubDocumentType,
  PropType,
} from "@typegoose/typegoose";
import {
  User as DiscordUser,
  Role as DiscordRole,
  CommandInteraction,
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle,
  EmbedBuilder,
  Colors,
  ButtonInteraction,
  UserContextMenuCommandInteraction,
  MessageContextMenuCommandInteraction,
} from "discord.js";
import { ServerModel } from "../../ServerModel.js";
import {
  AccessGateSubGroupApplicationCommandOptionType,
  AccessListBarrier,
  ButtonIDFormat,
  ServerModelSelectionSnowflakeType,
  TargetClass,
  TargetClassSingular,
  TargetType,
} from "../../typings.js";
import { capitalizeFirstLetter } from "../../../utils/casing.js";
import { replyOrFollowUp } from "../../../utils/others.js";
import { AccessSelection } from "./AccessGate.js";
import { Command } from "./Command.js";
import { Whitelist } from "./Whitelist.js";
import { CounterModel } from "../Counter.js";

/**
 * Blacklist class
 * Represents a blacklist in the system
 */

@pre<Blacklist>("save", async function (next) {
  try {
    const counter = await CounterModel.findByIdAndUpdate(
      { _id: "caseNumber" },
      { $inc: { seq: 1 } },
      { new: true, upsert: true }
    );
    this.caseNumber = counter.seq;
    next();
  } catch (error: any) {
    return next(error);
  }
})
@post<Blacklist>("save", function (doc: DocumentType<Blacklist>) {
  console.log("A blacklist document has been saved.", doc.toJSON());
})
export class Blacklist extends AccessSelection {
  @prop({ type: () => [Command], default: [] }, PropType.ARRAY)
  public commands!: ArraySubDocumentType<Command>[];

  @prop()
  public caseNumber?: number;

  public async checkIfExists(
    target: AccessGateSubGroupApplicationCommandOptionType,
    targetClassStr: TargetClass,
    commandName?: string
  ) {
    return commandName
      ? this.commands!.findIndex(
          (cmd) =>
            cmd.commandName === commandName &&
            cmd[targetClassStr]!.find((v) => v.id == target.id)
        ) != -1
      : typeof this[targetClassStr]!.find((v) => v.id == target.id) !=
          "undefined";
  }

  public async addToList(
    this: DocumentType<Blacklist>,
    element: ServerModelSelectionSnowflakeType,
    interaction: CommandInteraction | ButtonInteraction
  ) {
    const strProp = interaction.guild!.members.cache.has(element.id)
      ? "users"
      : interaction.guild!.roles.cache.has(element.id)
      ? "roles"
      : "channels";
    (this[strProp]! as ServerModelSelectionSnowflakeType[]).push(element);
    return await this.$parent()!.save();
  }

  public async removeFromList(
    this: DocumentType<Blacklist>,
    element: ServerModelSelectionSnowflakeType,
    interaction: CommandInteraction | ButtonInteraction
  ) {
    const strProp = interaction.guild!.members.cache.has(element.id)
      ? "users"
      : interaction.guild!.roles.cache.has(element.id)
      ? "roles"
      : "channels";
    const selection = this[strProp] as ServerModelSelectionSnowflakeType[];
    (this[strProp] as ServerModelSelectionSnowflakeType[]) = selection.filter(
      (e) => e.id != element.id
    );

    return await this.$parent()!.save();
  }

  public async applicationModifySelection(params: {
    type: AccessGateSubGroupApplicationCommandOptionType;
    interaction:
      | CommandInteraction
      | ButtonInteraction
      | UserContextMenuCommandInteraction
      | MessageContextMenuCommandInteraction;
    list: AccessListBarrier;
    action: "add" | "remove";
    commandName?: string;
    transfering?: boolean;
  }) {
    const { type, commandName, interaction, list, action, transfering } =
      params;
    console.log("🚀 ~ file: AccessGate.ts:163 ~ Blacklist ~ params:");

    const targetClassStr: TargetClass = interaction.guild?.members.cache.has(
      type.id
    )
      ? `users`
      : interaction.guild?.roles.cache.has(type.id)
      ? `roles`
      : `channels`;

    console.log(
      "🚀 ~ file: AccessGate.ts:174 ~ Blacklist ~ targetClassStr:",
      targetClassStr
    );

    const targetTypeStr = capitalizeFirstLetter(
      targetClassStr.slice(0, -1)
    ) as TargetType;

    const targetMention = `<${
      interaction.guild!.members.cache.has(type.id)
        ? "@"
        : interaction.guild!.roles.cache.has(type.id)
        ? "@&"
        : "#"
    }${type!.id}>`;

    let server = await ServerModel.findOne({
      serverId: interaction.guildId,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const targetObj = {
      id: type.id,
      name:
        interaction.guild!.members.cache.get(type.id)?.user.tag ??
        (type as DiscordRole).name,
    };

    const oppositeList = list === "whitelist" ? "blacklist" : "whitelist";

    const serverListObj = server.cases[list] as DocumentType<Blacklist>;
    const serverOppositeListObj = server.cases[
      oppositeList
    ] as DocumentType<Whitelist>;

    if (
      action == "add" &&
      (await this.checkIfExists(type, targetClassStr, commandName))
    ) {
      await replyOrFollowUp(interaction, {
        content: `${targetMention} is already in the ${list}.`,
        ephemeral: true,
      });
      return;
    }

    if (
      action == "remove" &&
      !(await serverListObj.checkIfExists(type, targetClassStr, commandName))
    ) {
      await replyOrFollowUp(interaction, {
        content: `${targetMention} does not exist in the ${list}.`,
        ephemeral: true,
      });
      return;
    }

    if (
      !transfering &&
      action == "add" &&
      (await serverOppositeListObj.checkIfExists(
        type,
        targetClassStr,
        commandName
      ))
    ) {
      const snowflakeSingular = targetClassStr.slice(
        0,
        -1
      ) as TargetClassSingular;
      const buttonIdPrefix: ButtonIDFormat = `${list}_${snowflakeSingular}_`;

      let row = new ActionRowBuilder<ButtonBuilder>().addComponents(
        new ButtonBuilder()
          .setCustomId(`${buttonIdPrefix}move_target`)
          .setLabel("Yes")
          .setStyle(ButtonStyle.Success),
        new ButtonBuilder()
          .setCustomId(`${buttonIdPrefix}cancel_move`)
          .setLabel("No")
          .setStyle(ButtonStyle.Danger)
      );
      console.log("#    🚀 ~ file: AccessGate.ts:253 ~ Blacklist ~ row:");

      let confirmationEmbed = new EmbedBuilder()
        .setTitle("Confirmation")
        .setDescription(
          `${targetMention} exists in the ${oppositeList} ${
            commandName ?? "guild"
          } database. Do you want to move this data to the ${list}?`
        )
        .setColor(Colors.Gold) // Yellow color for confirmation
        .setAuthor({
          name: targetObj.name,
        })
        .setFooter({
          text: `${targetTypeStr} ID: ${targetObj.id}`,
        });

      if (interaction.guild!.members.cache.has(type.id))
        confirmationEmbed.toJSON().author!.icon_url = (
          type as DiscordUser
        ).displayAvatarURL();
      await replyOrFollowUp(interaction, {
        embeds: [confirmationEmbed],
        ephemeral: true,
        components: [row],
      });

      return;
    } else {
      const functionStr = action == "add" ? "addToList" : "removeFromList";
      const embedDirectionStr = action == "add" ? "added to" : "removed from";
      await serverListObj[functionStr](targetObj, interaction);
      if (!transfering) {
        const successEmbed = new EmbedBuilder()
          .setTitle("Success")
          .setDescription(
            `${targetMention} has been ${embedDirectionStr} the ${list}`
          )
          .setColor(Colors.Green) // Green color for success
          .setAuthor({
            name: targetObj.name,
          })
          .setFooter({ text: `${targetTypeStr} ID: ${targetObj.id}` })
          .setTimestamp();

        if (interaction.guild!.members.cache.has(type.id))
          successEmbed.toJSON().author!.icon_url = (
            type as DiscordUser
          ).displayAvatarURL();

        await replyOrFollowUp(interaction, {
          embeds: [successEmbed],
          ephemeral: true,
        });

        return;
      }
    }
  }
}
```

## \src\models\Moderation\AccessGate\Command.ts
```ts
import { pre, post, prop, DocumentType } from "@typegoose/typegoose";
import { CommandInteraction, ButtonInteraction } from "discord.js";
import { ServerModelSelectionSnowflakeType } from "../../typings.js";
import { AccessSelection } from "./AccessGate.js";

/**
 * Command class
 * Represents a command in the system
 */

@pre<Command>("save", function (next) {
  console.log("A command document is going to be saved.");
  next();
})
@post<Command>("save", function (doc: DocumentType<Command>) {
  console.log("A command document has been saved.", doc.toJSON());
})
export class Command extends AccessSelection {
  @prop({ required: true })
  public commandName!: string;

  public async addToList(
    this: DocumentType<Command>,
    element: ServerModelSelectionSnowflakeType,
    interaction: CommandInteraction | ButtonInteraction
  ) {
    const strProp = interaction.guild!.members.cache.has(element.id)
      ? "users"
      : interaction.guild!.roles.cache.has(element.id)
      ? "roles"
      : "channels";

    (this[strProp] as ServerModelSelectionSnowflakeType[]).push(element);
    return await this.$parent()!.save();
  }

  public async removeFromList(
    this: DocumentType<Command>,
    element: ServerModelSelectionSnowflakeType,
    interaction: CommandInteraction | ButtonInteraction
  ) {
    const strProp = interaction.guild!.members.cache.has(element.id)
      ? "users"
      : interaction.guild!.roles.cache.has(element.id)
      ? "roles"
      : "channels";
    const selection = this[strProp] as ServerModelSelectionSnowflakeType[];

    (this[strProp] as ServerModelSelectionSnowflakeType[]) = selection.filter(
      (e) => e.id != element.id
    );

    return await this.$parent()!.save();
  }
}
```

## \src\models\Moderation\AccessGate\Whitelist.ts
```ts
import {
  pre,
  post,
  prop,
  DocumentType,
  ArraySubDocumentType,
  PropType,
} from "@typegoose/typegoose";
import {
  User as DiscordUser,
  Role as DiscordRole,
  CommandInteraction,
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle,
  EmbedBuilder,
  Colors,
  ButtonInteraction,
  UserContextMenuCommandInteraction,
  MessageContextMenuCommandInteraction,
} from "discord.js";
import { ServerModel } from "../../ServerModel.js";
import {
  AccessGateSubGroupApplicationCommandOptionType,
  AccessListBarrier,
  ServerModelSelectionSnowflakeType,
  TargetClass,
  TargetType,
} from "../../typings.js";
import { capitalizeFirstLetter } from "../../../utils/casing.js";
import { replyOrFollowUp } from "../../../utils/others.js";
import { Blacklist } from "./Blacklist.js";
import { AccessSelection } from "./AccessGate.js";
import { Command } from "./Command.js";
import { CounterModel } from "../Counter.js";

/**
 * Whitelist class
 * Represents a whitelist in the system
 */
@pre<Whitelist>("save", async function (next) {
  try {
    const counter = await CounterModel.findByIdAndUpdate(
      { _id: "caseNumber" },
      { $inc: { seq: 1 } },
      { new: true, upsert: true }
    );
    this.caseNumber = counter.seq;
    next();
  } catch (error: any) {
    return next(error);
  }
})
@post<Whitelist>("save", function (doc: DocumentType<Whitelist>) {
  console.log("A whitelist document has been saved.", doc.toJSON());
})
export class Whitelist extends AccessSelection {
  @prop({ type: () => [Command], default: [] }, PropType.ARRAY)
  public commands!: ArraySubDocumentType<Command>[];

  @prop()
  public caseNumber?: number;

  public async checkIfExists(
    target: AccessGateSubGroupApplicationCommandOptionType,
    targetClassStr: TargetClass,
    commandName?: string
  ) {
    return commandName
      ? this.commands!.findIndex(
          (cmd) =>
            cmd.commandName === commandName &&
            cmd[targetClassStr]!.find((v) => v.id == target.id)
        ) != -1
      : this[targetClassStr]!.find((v) => v.id == target.id);
  }

  public async addToList(
    this: DocumentType<Whitelist>,
    element: ServerModelSelectionSnowflakeType,
    interaction: CommandInteraction | ButtonInteraction
  ) {
    const strProp = interaction.guild!.members.cache.has(element.id)
      ? "users"
      : interaction.guild!.roles.cache.has(element.id)
      ? "roles"
      : "channels";
    (this[strProp]! as ServerModelSelectionSnowflakeType[]).push(element);
    return await this.$parent()!.save();
  }

  public async removeFromList(
    this: DocumentType<Whitelist>,
    element: ServerModelSelectionSnowflakeType,
    interaction: CommandInteraction | ButtonInteraction
  ) {
    const strProp = interaction.guild!.members.cache.has(element.id)
      ? "users"
      : interaction.guild!.roles.cache.has(element.id)
      ? "roles"
      : "channels";
    const selection = this[strProp] as ServerModelSelectionSnowflakeType[];
    (this[strProp] as ServerModelSelectionSnowflakeType[]) = selection.filter(
      (e) => e.id != element.id
    );

    return await this.$parent()!.save();
  }

  public async applicationModifySelection(params: {
    type: AccessGateSubGroupApplicationCommandOptionType;
    interaction:
      | CommandInteraction
      | ButtonInteraction
      | UserContextMenuCommandInteraction
      | MessageContextMenuCommandInteraction;
    list: AccessListBarrier;
    action: "add" | "remove";
    commandName?: string;
    transfering?: boolean;
  }) {
    const { type, commandName, interaction, list, action, transfering } =
      params;
    console.log("🚀 ~ file: AccessGate.ts:163 ~ Whitelist ~ params:");

    const targetClassStr: TargetClass = interaction.guild!.members.cache.has(
      type.id
    )
      ? `users`
      : interaction.guild!.roles.cache.has(type.id)
      ? `roles`
      : `channels`;

    const targetTypeStr = capitalizeFirstLetter(
      targetClassStr.slice(0, -1)
    ) as TargetType;

    const targetMention = `<${
      interaction.guild!.members.cache.has(type.id)
        ? "@"
        : interaction.guild!.roles.cache.has(type.id)
        ? "@&"
        : "#"
    }${type!.id}>`;

    let server = await ServerModel.findOne({
      serverId: interaction.guildId,
    });

    if (!server) {
      server = await new ServerModel({
        createdBy: {
          id: interaction.guild?.ownerId,
          name: (await interaction.guild?.fetchOwner())!.user.tag,
        },
        serverId: interaction.guildId,
        serverName: interaction.guild?.name,
      }).save();
    }

    const targetObj = {
      id: type.id,
      name: interaction.guild!.members.cache.has(type.id)
        ? (type as DiscordUser).tag
        : (type as DiscordRole).name,
    };

    const oppositeList = list === "whitelist" ? "blacklist" : "whitelist";

    const serverListObj = server.cases[list] as DocumentType<Whitelist>;
    const serverOppositeListObj = server.cases[
      oppositeList
    ] as DocumentType<Blacklist>;

    if (
      action == "add" &&
      (await this.checkIfExists(type, targetClassStr, commandName))
    ) {
      await replyOrFollowUp(interaction, {
        content: `${targetMention} is already in the ${list}.`,
        ephemeral: true,
      });
      return;
    }

    if (
      action == "remove" &&
      !(await this.checkIfExists(type, targetClassStr, commandName))
    ) {
      await replyOrFollowUp(interaction, {
        content: `${targetMention} does not exist in the ${list}.`,
        ephemeral: true,
      });

      return;
    }

    if (
      !transfering &&
      action == "add" &&
      (await serverOppositeListObj.checkIfExists(
        type,
        targetClassStr,
        commandName
      ))
    ) {
      const buttonIdPrefix = `${list}_${targetClassStr.slice(0, -1)}`;

      let row = new ActionRowBuilder<ButtonBuilder>().addComponents(
        new ButtonBuilder()
          .setCustomId(`${buttonIdPrefix}_move_target`)
          .setLabel("Yes")
          .setStyle(ButtonStyle.Success),
        new ButtonBuilder()
          .setCustomId(`${buttonIdPrefix}_cancel_move`)
          .setLabel("No")
          .setStyle(ButtonStyle.Danger)
      );
      console.log("#    🚀 ~ file: AccessGate.ts:472 ~ Whitelist ~ row:");

      let confirmationEmbed = new EmbedBuilder()
        .setTitle("Confirmation")
        .setDescription(
          `${targetMention} exists in the ${oppositeList} ${
            commandName ?? "guild"
          } database. Do you want to move this data to the ${list}?`
        )
        .setColor(Colors.Gold) // Yellow color for confirmation
        .setAuthor({
          name: targetObj.name,
        })
        .setFooter({
          text: `${targetTypeStr} ID: ${targetObj.id}`,
        });

      if (interaction.guild!.members.cache.has(type.id))
        confirmationEmbed.toJSON().author!.icon_url = (
          type as DiscordUser
        ).displayAvatarURL();

      await replyOrFollowUp(interaction, {
        embeds: [confirmationEmbed],
        ephemeral: true,
        components: [row],
      });

      return;
    } else {
      const functionStr = action == "add" ? "addToList" : "removeFromList";
      const embedDirectionStr = action == "add" ? "added to" : "removed from";

      await serverListObj[functionStr](targetObj, interaction);
      if (!transfering) {
        const successEmbed = new EmbedBuilder()
          .setTitle("Success")
          .setDescription(
            `${targetMention} has been ${embedDirectionStr} the ${list}`
          )
          .setColor(Colors.Green) // Green color for success
          .setAuthor({
            name: targetObj.name,
          })
          .setFooter({ text: `${targetTypeStr} ID: ${targetObj.id}` })
          .setTimestamp();

        if (interaction.guild!.members.cache.has(type.id))
          successEmbed.toJSON().author!.icon_url = (
            type as DiscordUser
          ).displayAvatarURL();

        await replyOrFollowUp(interaction, {
          embeds: [successEmbed],
          ephemeral: true,
        });

        return;
      }
    }
  }
}
```

## \src\utils\casing.ts
```ts
export function capitalizeFirstLetter(str: string) {
  return str.charAt(0).toUpperCase() + str.slice(1);
}
```

## \src\utils\defaults.ts
```ts
export default {
  cooldown: 3000,
};
```

## \src\utils\disableButtons.ts
```ts
import { ActionRowBuilder, ButtonBuilder } from "@discordjs/builders";

export const disableButtons = (rows: ActionRowBuilder<ButtonBuilder>[]) => {
  rows.forEach((row) => row.components.map((r) => r.setDisabled(true)));

  return rows;
};
```

## \src\utils\getEfficientTimeDifference.ts
```ts
export function getEfficientTimeDifference(date1: Date, date2?: Date): string {
  if (!date2) date2 = new Date();

  const diffMs = Math.abs(date2.getTime() - date1.getTime());
  const diffSec = diffMs / 1000;
  const diffMin = diffSec / 60;
  const diffHours = diffMin / 60;
  const diffDays = diffHours / 24;

  if (diffSec < 1) {
    return `${diffMs}ms`;
  } else if (diffMin < 1) {
    return `${Math.floor(diffSec)}s`;
  } else if (diffHours < 1) {
    return `${Math.floor(diffMin)}m${Math.floor(diffSec % 60)}s`;
  } else if (diffDays < 1) {
    return `${Math.floor(diffHours)}h${Math.floor(diffMin % 60)}m`;
  } else {
    return `${Math.floor(diffDays)}d${Math.floor(diffHours % 24)}h`;
  }
}
```

## \src\utils\moderationHierachy.ts
```ts
import { ModerationHierachy } from "./types.js";
import {
  User,
  Role,
  CommandInteraction,
  GuildMember,
  UserContextMenuCommandInteraction,
  MessageContextMenuCommandInteraction,
} from "discord.js";

async function checkRoleHierarchy(
  interactionAuthor: GuildMember,
  interactionTarget: GuildMember | Role
): Promise<ModerationHierachy | void> {
  if (interactionTarget instanceof GuildMember && interactionTarget.user.bot) {
    return "You cannot select a bot";
  }

  const targetPosition =
    interactionTarget instanceof GuildMember
      ? interactionTarget.roles.highest.position
      : interactionTarget.position;

  if (targetPosition >= interactionAuthor.roles.highest.position) {
    return `You cannot select that ${
      interactionTarget instanceof GuildMember ? "user" : "role"
    } as they are higher or equal to your target in the role hierachy`;
  }
}

export async function moderationHierachy(
  target: User | Role,
  interaction:
    | CommandInteraction
    | UserContextMenuCommandInteraction
    | MessageContextMenuCommandInteraction
): Promise<ModerationHierachy | void> {
  const interactionAuthor = interaction.guild?.members.cache.get(
    interaction.user.id
  )!;

  if (target.id == interaction.user.id) return "You cannot select yourself";

  const guildMember = await interaction.guild?.members.fetch(target.id);
  const interactionTarget = interaction.guild?.[
    guildMember instanceof GuildMember ? "members" : "roles"
  ].cache.get(target.id)!;

  return checkRoleHierarchy(interactionAuthor, interactionTarget);
}
```

## \src\utils\others.ts
```ts
import { FilteredKeys, MongooseDocType } from "../models/typings";
import {
  SubDocumentType,
  DocumentType,
  ReturnModelType,
} from "@typegoose/typegoose";
import {
  AnyParamConstructor,
  BeAnObject,
  IObjectWithTypegooseFunction,
} from "@typegoose/typegoose/lib/types";
import {
  CommandInteraction,
  MessageComponentInteraction,
  InteractionReplyOptions,
} from "discord.js";
import { Document, Types } from "mongoose";

export async function replyOrFollowUp(
  interaction: CommandInteraction | MessageComponentInteraction,
  replyOptions:
    | (InteractionReplyOptions & {
        ephemeral?: boolean;
      })
    | string
) {
  // if interaction is already replied
  if (interaction.replied) {
    await interaction.followUp(replyOptions);
    return;
  }

  // if interaction is deferred but not replied
  if (interaction.deferred) {
    await interaction.editReply(replyOptions);
    return;
  }

  // if interaction is not handled yet
  await interaction.reply(replyOptions);
  return;
}

export function typegooseClassProps<T extends object>(obj: T) {
  let result: {
    [key: string]: any;
  } = {};

  for (const key in obj) {
    const typedKey = key as keyof T;

    if (!key.startsWith("_") && typeof obj[typedKey] !== "function") {
      result[typedKey as any] = obj[typedKey];
    }
  }

  return result as Omit<FilteredKeys<T>, "_id">;
}
```

## \src\utils\PaginationButtons.ts
```ts
import { Blacklist } from "../models/Moderation/AccessGate/Blacklist.js";
import { Command } from "../models/Moderation/AccessGate/Command.js";
import { Whitelist } from "../models/Moderation/AccessGate/Whitelist.js";
import { ModerationCases } from "../models/Moderation/ModerationCases.js";
import { Server, ServerModel } from "../models/ServerModel.js";
import {
  AccessListBarrier,
  ButtonIDFormat,
  MongooseDocType,
  ServerModelSelectionSnowflakeType,
  TargetClass,
  TargetClassSingular,
  TargetType,
} from "../models/typings.js";
import { ClassPropertyNames } from "./types.js";
import {
  Pagination,
  PaginationOptions,
  PaginationType,
} from "@discordx/pagination";
import { DocumentType } from "@typegoose/typegoose";
import {
  ButtonInteraction,
  ButtonStyle,
  Colors,
  EmbedBuilder,
  Role as DiscordRole,
  GuildMember,
  GuildBasedChannel,
  CommandInteraction,
} from "discord.js";

export class PaginationButtons {}

export async function paginateData(
  data: (Command | ServerModelSelectionSnowflakeType)[],
  interaction: CommandInteraction
): Promise<string[]> {
  const maxEmbedDescriptionLength = 2048;
  const descriptions: string[] = [];

  const getMention = async (snowflake: ServerModelSelectionSnowflakeType) => {
    if (interaction.guild!.channels.cache.has(snowflake.id))
      return `<#${snowflake.id}>`;
    else if (interaction.guild!.roles.cache.has(snowflake.id))
      return `<@&${snowflake.id}>`;

    try {
      return (
        (await interaction.guild!.members.fetch(snowflake.id)) ||
        (await interaction.guild!.channels.fetch(snowflake.id)) ||
        (await interaction.guild!.roles.fetch(snowflake.id))
      )?.toString();
    } catch (error) {
      return `undefined`;
    }
  };

  let listIndex = 1;

  while (data && data.length > 0) {
    let description = "";
    let index = 0;
    let info = "";

    while (index < data.length) {
      if (data[index] instanceof Command) {
        const command = data[index] as Command;

        const { commandName, users, roles, channels } = command;

        for (const group of [users, roles, channels].filter(
          async (v, i) => typeof (await getMention(v[i])) != "undefined"
        )) {
          for (const item of group) {
            info = `\`${commandName}\`: ${item.name} (${item.id})`;
            const entry = `\`${listIndex}\` ${info}\n`;

            if (description.length + entry.length > maxEmbedDescriptionLength) {
              break;
            }

            description += entry;
            listIndex++;
            index++;
          }
        }
      } else {
        const snowflake = data[index] as ServerModelSelectionSnowflakeType;

        info = await getMention(snowflake);

        const entry = `\`${listIndex}\` ${info}\n`;

        if (description.length + entry.length > maxEmbedDescriptionLength) {
          break;
        }

        description += entry;
        listIndex++;
        index++;
      }

      if (description.length + info.length > maxEmbedDescriptionLength) {
        break;
      }
    }

    descriptions.push(description);
    data = data.slice(index);
  }

  return descriptions;
}

export function paginationButtonsRow(
  list: AccessListBarrier,
  snowflakePlural: TargetClass | "guilds" = "guilds"
) {
  const snowflakeSingular = snowflakePlural.slice(0, -1) as TargetClassSingular;

  const prefix: ButtonIDFormat<"pagination"> = `${list}_${snowflakeSingular}_pagination`;

  const positionData: PaginationOptions = {
    type: PaginationType.Button,
    start: {
      emoji: {
        name: "⏮️",
      },
      id: `${prefix}_beginning`,
      label: " ",
      style: ButtonStyle.Secondary,
    },
    previous: {
      emoji: {
        name: "⬅️",
      },
      id: `${prefix}_previous`,
      label: " ",
      style: ButtonStyle.Secondary,
    },
    next: {
      emoji: {
        name: "➡️",
      },
      id: `${prefix}_next`,
      label: " ",
      style: ButtonStyle.Secondary,
    },

    end: {
      emoji: {
        name: "⏭️",
      },
      id: `${prefix}_end`,
      label: " ",
      style: ButtonStyle.Secondary,
    },
    ephemeral: true,
  };

  return positionData;
}

export async function ButtonComponentMoveSnowflake(
  interaction: ButtonInteraction
) {
  await interaction.deferReply({ ephemeral: true });
  const server = (await ServerModel.findOne({
    serverId: interaction.guild?.id!,
  }))!;

  const fetchedMessage = interaction.message;
  const confirmationEmbed = fetchedMessage.embeds[0];
  const messageContentArray = confirmationEmbed.description!.split(" ");
  const footerWordArr = confirmationEmbed.footer!.text.split(" ");

  let commandName: string | undefined;

  if (messageContentArray.indexOf("guild") == -1)
    commandName =
      messageContentArray[messageContentArray.indexOf("database") - 1];

  const targetTypeStr = footerWordArr[0] as TargetType;
  const targetGuildPropertyStr =
    targetTypeStr == `User`
      ? `members`
      : (`${targetTypeStr.toLowerCase()}s` as `roles` | `channels`);

  const target = (await interaction.guild?.[targetGuildPropertyStr].fetch(
    confirmationEmbed.footer!.text!.split(" ").at(-1)!
  ))!;

  const targetMention = `<${
    interaction.guild!.members.cache.has(target.id)
      ? "@"
      : interaction.guild!.roles.cache.has(target.id)
      ? "@&"
      : "#"
  }${target!.id}>`;

  const list = messageContentArray.pop()?.slice(0, -1) as AccessListBarrier; // do you want to move this data to the [whitelist | blacklist] -> the one to add to database

  const listInstance = server.cases[list] as DocumentType<
    Blacklist | Whitelist
  >;

  const oppositeList = list === "whitelist" ? "blacklist" : "whitelist"; // the one to remove from the database

  const oppositeListInstance = server.cases[
    oppositeList
  ] as typeof listInstance;

  await oppositeListInstance.applicationModifySelection({
    action: "remove",
    commandName,
    interaction,
    type: interaction.guild!.members.cache.has(target.id)
      ? (target as GuildMember).user
      : (target as DiscordRole | GuildBasedChannel),
    list: oppositeList,
    transfering: true,
  });

  // in case the bot shutdown unexpectedly, it's better to remove the data first than to have the target in both whitelist and blacklsit

  await listInstance.applicationModifySelection({
    action: "add",
    commandName,
    interaction,
    type: interaction.guild!.members.cache.has(target.id)
      ? (target as GuildMember).user
      : (target as DiscordRole | GuildBasedChannel),
    list,
    transfering: true,
  });

  let confirmedEmbed = new EmbedBuilder()
    .setTitle("Success")
    .setDescription(
      `${targetMention} has been moved from the ${oppositeList} to the ${list}  ${
        commandName ?? "guild"
      }`
    )
    .setColor(Colors.Green)
    .setAuthor(confirmationEmbed.author)
    .setFooter(confirmationEmbed.footer)
    .setTimestamp();

  await interaction.editReply({
    embeds: [confirmedEmbed],
    components: [],
  });
}

export async function PaginationSender(params: {
  server: MongooseDocType<Server>;
  list: ClassPropertyNames<ModerationCases>;
  snowflakePluralType: TargetClass | "guilds";
  interaction: CommandInteraction;
  commandName?: string;
}) {
  const { server, list, snowflakePluralType, commandName, interaction } =
    params;

  let data: string[];
  if (commandName)
    data = await paginateData(
      server.cases[list].commands.filter((v) => (v.commandName = commandName)),
      interaction
    );
  else
    data = await paginateData(
      snowflakePluralType == "guilds"
        ? [
            ...server.cases[list].channels,
            ...server.cases[list].roles,
            ...server.cases[list].users,
          ]
        : server.cases[list][snowflakePluralType],
      interaction
    );

  if (data.length == 0)
    return await interaction.reply({
      content: `Nothing to view yet in this query selection.`,
      ephemeral: true,
    });

  let pages = data.map((d, i) => {
    return {
      embeds: [
        new EmbedBuilder()
          .setTitle(`${interaction.guild!.name} Cases`)
          .setDescription(d)
          .setColor(Colors.Gold)
          .setFooter({
            text: `Page ${i + 1}/${data.length}`,
          }),
      ],
    };
  });

  const buttons = paginationButtonsRow(list, snowflakePluralType);

  return await new Pagination(interaction, pages, buttons).send();
}
```

## \src\utils\registry.ts
```ts
import { glob } from "glob";
import path from "path";
import { fileURLToPath } from "url";

export const isESM = true;

export function dirname(url: string): string {
  return path.dirname(fileURLToPath(url));
}

export async function resolve(...paths: string[]): Promise<string[]> {
  const imports: string[] = [];

  await Promise.all(
    paths.map(async (ps) => {
      const files = await glob(ps.split(path.sep).join("/"));

      files.forEach((file) => {
        if (!imports.includes(file)) {
          imports.push("file://" + file);
        }
      });
    })
  );

  return imports;
}

export async function importx(...paths: string[]): Promise<void> {
  const files = await resolve(...paths);
  await Promise.all(files.map((file) => import(file)));
}

// await importx(`${dirname(import.meta.url)}/{events,commands}/**/*.{ts,js}`);
```

## \src\utils\types.ts
```ts
import { Server } from "../models/ServerModel";

export type FunctionPropertyNames<T> = {
  [K in keyof T]: T[K] extends Function ? K : never;
}[keyof T];
export type FunctionProperties<T> = Pick<T, FunctionPropertyNames<T>>;

export type NonFunctionPropertyNames<T> = {
  [K in keyof T]: T[K] extends Function ? never : K;
}[keyof T];
export type NonFunctionProperties<T> = Pick<T, NonFunctionPropertyNames<T>>;

export type ModerationHierachy =
  | `You cannot select yourself`
  | `You cannot select a bot`
  | `You cannot select that ${
      | `user`
      | `role`} as they are higher or equal to your target in the role hierachy`;

export type ClassPropertyNames<T> = {
  [K in keyof T]: T[K] extends Function ? never : K;
}[keyof T] extends infer U
  ? U
  : never;
```
.
