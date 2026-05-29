const {
  Client,
  GatewayIntentBits,
  Partials,
  SlashCommandBuilder,
  Routes,
  REST,
  PermissionFlagsBits,
  EmbedBuilder,
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle,
  ModalBuilder,
  TextInputBuilder,
  TextInputStyle,
  ChannelType
} = require('discord.js');

const fs = require('fs');

const TOKEN = process.env.TOKEN;
const CLIENT_ID = process.env.CLIENT_ID;

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.DirectMessages,
    GatewayIntentBits.GuildMembers
  ],
  partials: [Partials.Channel]
});

const dbFile = './database.json';

if (!fs.existsSync(dbFile)) {
  fs.writeFileSync(dbFile, JSON.stringify({
    logs: {},
    appLogs: {},
    applications: {}
  }, null, 2));
}

function loadDB() {
  return JSON.parse(fs.readFileSync(dbFile));
}

function saveDB(data) {
  fs.writeFileSync(dbFile, JSON.stringify(data, null, 2));
}

const commands = [
  new SlashCommandBuilder()
    .setName('kick')
    .setDescription('Kick a member')
    .addUserOption(option =>
      option.setName('user').setDescription('User').setRequired(true))
    .addStringOption(option =>
      option.setName('reason').setDescription('Reason').setRequired(false))
    .setDefaultMemberPermissions(PermissionFlagsBits.Administrator),

  new SlashCommandBuilder()
    .setName('ban')
    .setDescription('Ban a member')
    .addUserOption(option =>
      option.setName('user').setDescription('User').setRequired(true))
    .addStringOption(option =>
      option.setName('time').setDescription('Ban duration').setRequired(false))
    .addStringOption(option =>
      option.setName('reason').setDescription('Reason').setRequired(false))
    .setDefaultMemberPermissions(PermissionFlagsBits.Administrator),

  new SlashCommandBuilder()
    .setName('mute')
    .setDescription('Timeout a member')
    .addUserOption(option =>
      option.setName('user').setDescription('User').setRequired(true))
    .addIntegerOption(option =>
      option.setName('minutes').setDescription('Minutes').setRequired(true))
    .addStringOption(option =>
      option.setName('reason').setDescription('Reason').setRequired(false))
    .setDefaultMemberPermissions(PermissionFlagsBits.Administrator),

  new SlashCommandBuilder()
    .setName('poll')
    .setDescription('Create a poll')
    .addChannelOption(option =>
      option.setName('channel')
        .setDescription('Poll channel')
        .addChannelTypes(ChannelType.GuildText)
        .setRequired(true))
    .addStringOption(option =>
      option.setName('question').setDescription('Question').setRequired(true))
    .addStringOption(option =>
      option.setName('choices').setDescription('Use | to separate').setRequired(true)),

  new SlashCommandBuilder()
    .setName('setlog')
    .setDescription('Set moderation logs')
    .addChannelOption(option =>
      option.setName('channel')
        .setDescription('Log channel')
        .addChannelTypes(ChannelType.GuildText)
        .setRequired(true))
    .setDefaultMemberPermissions(PermissionFlagsBits.Administrator),

  new SlashCommandBuilder()
    .setName('setapplog')
    .setDescription('Set application logs')
    .addChannelOption(option =>
      option.setName('channel')
        .setDescription('Application log channel')
        .addChannelTypes(ChannelType.GuildText)
        .setRequired(true))
    .setDefaultMemberPermissions(PermissionFlagsBits.Administrator),

  new SlashCommandBuilder()
    .setName('announce')
    .setDescription('Send an announcement')
    .addChannelOption(option =>
      option.setName('channel')
        .setDescription('Announcement channel')
        .addChannelTypes(ChannelType.GuildText)
        .setRequired(true))
    .addStringOption(option =>
      option.setName('message')
        .setDescription('Announcement')
        .setRequired(true))
    .setDefaultMemberPermissions(PermissionFlagsBits.Administrator),

  new SlashCommandBuilder()
    .setName('appconfig')
    .setDescription('Open the application config panel')
    .setDefaultMemberPermissions(PermissionFlagsBits.Administrator),

  new SlashCommandBuilder()
    .setName('apply')
    .setDescription('Apply for an application')
    .addStringOption(option =>
      option.setName('application')
        .setDescription('Application name')
        .setRequired(true))
].map(cmd => cmd.toJSON());

client.once('ready', async () => {
  console.log(`🧭 Logged in as ${client.user.tag}`);

  const rest = new REST({ version: '10' }).setToken(TOKEN);

  try {
    await rest.put(
      Routes.applicationCommands(CLIENT_ID),
      { body: commands }
    );

    console.log('✅ Slash commands registered');
  } catch (err) {
    console.error(err);
  }
});

client.on('interactionCreate', async interaction => {

  if (interaction.isChatInputCommand()) {

    const db = loadDB();

    if (interaction.commandName === 'kick') {
      const user = interaction.options.getUser('user');
      const reason = interaction.options.getString('reason') || 'No reason';

      const member = await interaction.guild.members.fetch(user.id);

      await member.kick(reason);

      await interaction.reply(`🧭 Kicked ${user.tag}`);
    }

    if (interaction.commandName === 'ban') {
      const user = interaction.options.getUser('user');
      const reason = interaction.options.getString('reason') || 'No reason';

      await interaction.guild.members.ban(user.id, { reason });

      await interaction.reply(`🔨 Banned ${user.tag}`);
    }

    if (interaction.commandName === 'mute') {
      const user = interaction.options.getUser('user');
      const minutes = interaction.options.getInteger('minutes');
      const reason = interaction.options.getString('reason') || 'No reason';

      const member = await interaction.guild.members.fetch(user.id);

      await member.timeout(minutes * 60 * 1000, reason);

      await interaction.reply(`🔇 Timed out ${user.tag}`);
    }

    if (interaction.commandName === 'poll') {

      const channel = interaction.options.getChannel('channel');
      const question = interaction.options.getString('question');
      const choices = interaction.options.getString('choices').split('|');

      const row = new ActionRowBuilder();

      choices.slice(0, 5).forEach((choice, index) => {
        row.addComponents(
          new ButtonBuilder()
            .setCustomId(`poll_${index}`)
            .setLabel(choice.trim())
            .setStyle(ButtonStyle.Primary)
        );
      });

      const embed = new EmbedBuilder()
        .setTitle('📊 Poll')
        .setDescription(question);

      await channel.send({
        embeds: [embed],
        components: [row]
      });

      await interaction.reply({
        content: '✅ Poll created',
        ephemeral: true
      });
    }

    if (interaction.commandName === 'setlog') {

      const channel = interaction.options.getChannel('channel');

      db.logs[interaction.guild.id] = channel.id;
      saveDB(db);

      await interaction.reply(`📖 Log channel set to ${channel}`);
    }

    if (interaction.commandName === 'setapplog') {

      const channel = interaction.options.getChannel('channel');

      db.appLogs[interaction.guild.id] = channel.id;
      saveDB(db);

      await interaction.reply(`📨 Application logs set to ${channel}`);
    }

    if (interaction.commandName === 'announce') {

      const channel = interaction.options.getChannel('channel');
      const message = interaction.options.getString('message');

      const embed = new EmbedBuilder()
        .setTitle('📢 Announcement')
        .setDescription(message);

      await channel.send({ embeds: [embed] });

      await interaction.reply({
        content: '✅ Announcement sent',
        ephemeral: true
      });
    }

    if (interaction.commandName === 'appconfig') {

      const row = new ActionRowBuilder()
        .addComponents(
          new ButtonBuilder()
            .setCustomId('create_app')
            .setLabel('Create Application')
            .setStyle(ButtonStyle.Success)
        );

      const embed = new EmbedBuilder()
        .setTitle('🧭 Compass Bot Application Config')
        .setDescription('Manage applications and questions.');

      await interaction.reply({
        embeds: [embed],
        components: [row],
        ephemeral: true
      });
    }

    if (interaction.commandName === 'apply') {

      const appName = interaction.options.getString('application');

      const modal = new ModalBuilder()
        .setCustomId(`application_modal_${appName}`)
        .setTitle(`${appName} Application`);

      const q1 = new TextInputBuilder()
        .setCustomId('question1')
        .setLabel('Why do you want this position?')
        .setStyle(TextInputStyle.Paragraph);

      const q2 = new TextInputBuilder()
        .setCustomId('question2')
        .setLabel('What experience do you have?')
        .setStyle(TextInputStyle.Paragraph);

      modal.addComponents(
        new ActionRowBuilder().addComponents(q1),
        new ActionRowBuilder().addComponents(q2)
      );

      await interaction.showModal(modal);
    }
  }

  if (interaction.isButton()) {

    if (interaction.customId === 'create_app') {

      const modal = new ModalBuilder()
        .setCustomId('create_application_modal')
        .setTitle('Create Application');

      const input = new TextInputBuilder()
        .setCustomId('application_name')
        .setLabel('Application Name')
        .setStyle(TextInputStyle.Short);

      modal.addComponents(
        new ActionRowBuilder().addComponents(input)
      );

      await interaction.showModal(modal);
    }
  }

  if (interaction.isModalSubmit()) {

    const db = loadDB();

    if (interaction.customId === 'create_application_modal') {

      const appName = interaction.fields.getTextInputValue('application_name');

      if (!db.applications[interaction.guild.id]) {
        db.applications[interaction.guild.id] = [];
      }

      db.applications[interaction.guild.id].push({
        name: appName
      });

      saveDB(db);

      await interaction.reply({
        content: `✅ Application "${appName}" created.`,
        ephemeral: true
      });
    }

    if (interaction.customId.startsWith('application_modal_')) {

      const appName = interaction.customId.replace('application_modal_', '');

      const answer1 = interaction.fields.getTextInputValue('question1');
      const answer2 = interaction.fields.getTextInputValue('question2');

      const logId = db.appLogs[interaction.guild.id];

      if (logId) {

        const channel = interaction.guild.channels.cache.get(logId);

        if (channel) {

          const embed = new EmbedBuilder()
            .setTitle(`📨 ${appName} Application`)
            .addFields(
              {
                name: 'Applicant',
                value: interaction.user.tag
              },
              {
                name: 'Why do you want this position?',
                value: answer1
              },
              {
                name: 'Experience',
                value: answer2
              }
            );

          await channel.send({
            embeds: [embed]
          });
        }
      }

      await interaction.reply({
        content: '✅ Application submitted.',
        ephemeral: true
      });
    }
  }
});

client.login(TOKEN);