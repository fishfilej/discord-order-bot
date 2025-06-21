const {
  Client,
  GatewayIntentBits,
  SlashCommandBuilder,
  ActionRowBuilder,
  StringSelectMenuBuilder,
  MessageFlags
} = require('discord.js');
const { REST } = require('@discordjs/rest');
const { Routes } = require('discord-api-types/v10');
require('dotenv').config();

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent
  ]
});

// In-memory storage for orders
const orders = new Map();
let orderCounter = 1;

// Slash command definition
const commands = [
  new SlashCommandBuilder()
    .setName('queue')
    .setDescription('Add a user to the order queue')
    .addUserOption(option =>
      option.setName('user').setDescription('The user to add to the queue').setRequired(true))
    .addChannelOption(option =>
      option.setName('channel').setDescription('The channel for the order').setRequired(true))
    .addStringOption(option =>
      option.setName('quantity').setDescription('Quantity').setRequired(true))
    .addStringOption(option =>
      option.setName('item').setDescription('Item name').setRequired(true))
    .addStringOption(option =>
      option.setName('duration').setDescription('Expected duration').setRequired(true))
]
  .map(command => command.toJSON());

// Deploy commands to Discord
const rest = new REST({ version: '10' }).setToken(process.env.TOKEN);
rest.put(
  Routes.applicationCommands(process.env.CLIENT_ID),
  { body: commands }
).then(() => {
  console.log('✅ Slash command registered.');
}).catch(console.error);

// Order message format
function createOrderMessage(order) {
  return (
    `⠀⠀⠀ **並ぶ,** <@${order.userId}> ❜ <#${order.channelId}>\n` +
    `⠀⠀⠀・ ${order.quantity}.　__${order.item}__\n` +
    `⠀⠀⠀・ duration — ${order.duration}\n` +
    `⠀⠀⠀・ 状態 ❜ **${order.status}**`
  );
}

// Create dropdown for status selection
function createStatusSelectMenu(orderId, currentStatus) {
  const selectMenu = new StringSelectMenuBuilder()
    .setCustomId(`status_update_${orderId}`)
    .setPlaceholder('order status');

  if (currentStatus === 'completed' || currentStatus === 'cancelled') {
    selectMenu.setDisabled(true).addOptions([
      { label: currentStatus, value: currentStatus }
    ]);
  } else {
    selectMenu.addOptions([
      { label: 'noted', value: 'noted' },
      { label: 'processing', value: 'processing' },
      { label: 'completed', value: 'completed' },
      { label: 'cancelled', value: 'cancelled' }
    ]);
  }

  return selectMenu;
}

// Interaction handler
client.on('interactionCreate', async interaction => {
  if (interaction.isChatInputCommand()) {
    if (interaction.commandName === 'queue') {
      try {
        const user = interaction.options.getUser('user');
        const channel = interaction.options.getChannel('channel');
        const quantity = interaction.options.getString('quantity');
        const item = interaction.options.getString('item');
        const duration = interaction.options.getString('duration');

        const order = {
          id: orderCounter++,
          userId: user.id,
          channelId: channel.id,
          quantity,
          item,
          duration,
          status: 'noted'
        };

        orders.set(order.id, order);

        const message = createOrderMessage(order);
        const selectMenu = createStatusSelectMenu(order.id, order.status);
        const row = new ActionRowBuilder().addComponents(selectMenu);

        await interaction.channel.send({
          content: message,
          components: [row],
          allowedMentions: { users: [], roles: [], everyone: false }
        });

        await interaction.reply({ content: '✅ Order created!', ephemeral: true });
      } catch (err) {
        console.error(err);
        await interaction.reply({ content: '❌ Failed to create order.', ephemeral: true });
      }
    }
  }

  // Handle status dropdown changes
  if (interaction.isStringSelectMenu()) {
    const [_, orderIdStr] = interaction.customId.split('_');
    const orderId = parseInt(orderIdStr);

    const order = orders.get(orderId);
    if (!order) return interaction.reply({ content: '❌ Order not found.', ephemeral: true });

    order.status = interaction.values[0]; // update status
    const updatedMessage = createOrderMessage(order);
    const updatedMenu = createStatusSelectMenu(order.id, order.status);
    const row = new ActionRowBuilder().addComponents(updatedMenu);

    await interaction.update({
      content: updatedMessage,
      components: [row]
    });
  }
});

// Bot ready
client.once('ready', () => {
  console.log(`🤖 Logged in as ${client.user.tag}`);
});

client.login(process.env.TOKEN);
