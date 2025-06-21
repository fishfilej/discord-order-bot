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

// In-memory storage for orders
const orders = new Map();
let orderCounter = 1;

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent
  ]
});

// Slash command setup
const commands = [
  new SlashCommandBuilder()
    .setName('queue')
    .setDescription('Add a user to the order queue')
    .addUserOption(option => 
      option.setName('user')
        .setDescription('The user to add to the queue')
        .setRequired(true))
    .addChannelOption(option => 
      option.setName('channel')
        .setDescription('The channel for the order')
        .setRequired(true))
    .addStringOption(option =>
      option.setName('quantity')
        .setDescription('Quantity as string')
        .setRequired(true))
    .addStringOption(option =>
      option.setName('item')
        .setDescription('Item name as string')
        .setRequired(true))
    .addStringOption(option =>
      option.setName('duration')
        .setDescription('Expected duration as string')
        .setRequired(true))
];

// Format the message
function createOrderMessage(order) {
  return (
    `⠀⠀⠀ **並ぶ,** <@${order.userId}> ❜ <#${order.channelId}>\n` +
    `⠀⠀⠀・ ${order.quantity}.　__${order.item}__\n` +
    `⠀⠀⠀・ duration — ${order.duration}\n` +
    `⠀⠀⠀・ 状態 ❜ **${order.status}**`
  );
}

function createStatusSelectMenu(orderId, currentStatus) {
  const selectMenu = new StringSelectMenuBuilder()
    .setCustomId(`status_update_${orderId}`)
    .setPlaceholder('order status');

  // If status is completed or cancelled, disable the dropdown
  if (currentStatus === 'completed' || currentStatus === 'cancelled') {
    selectMenu.setDisabled(true);
    selectMenu.addOptions([
      { label: currentStatus, value: currentStatus }
    ]);
  } else {
    // Normal options for active orders
    selectMenu.addOptions([
      { label: 'noted', value: 'noted' },
      { label: 'processing', value: 'processing' },
      { label: 'completed', value: 'completed' },
      { label: 'cancelled', value: 'cancelled' }
    ]);
  }

  return selectMenu;
}

// Slash command and select menu handler
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

        // Send message directly from the bot
        await interaction.channel.send({
          content: message,
          components: [row],
          allowedMentions: { users: [], roles: [], everyone: false }
        });
        
        //
