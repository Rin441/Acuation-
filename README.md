const { Client, GatewayIntentBits, EmbedBuilder } = require('discord.js');
const client = new Client({
    intents: [GatewayIntentBits.Guilds, GatewayIntentBits.GuildMessages, GatewayIntentBits.MessageContent]
});

const token = '';
const auctions = new Map();

const allowedAuctionChannels = [
    '1367449629733421068',
    '1367449640085094511',
    '1367449644971331645',
    '1367449649669214268',
    '1367449655272804403'
];

function formatTime(seconds) {
    const mins = Math.floor(seconds / 60);
    const secs = seconds % 60;
    return `${mins} دقيقة و ${secs} ثانية`;
}

client.once('ready', () => {
    console.log(`Logged in as ${client.user.tag}`);
});

client.on('messageCreate', async message => {
    if (message.author.bot) return;

    const content = message.content.trim();

    if (content === '$دليل') {
        if (!allowedAuctionChannels.includes(message.channel.id)) return;

        const embed = new EmbedBuilder()
            .setTitle('📘 دليل استخدام بوت المزاد')
            .setDescription(`
🔹 لبدء مزاد:
\`$مزاد <المدة> | <اسم العنصر>\`

مثال:
\`$مزاد 5 دقايق | حساب مميز\`

🔹 اكتب الرقم فقط للمزايدة.
🔹 يمكنك إرفاق صور في رسالة المزاد.
🔹 الروم يغلق تلقائيًا بعد انتهاء المزاد.
🔹 يتم فتح الروم تلقائيًا عند بدء مزاد جديد.
            `)
            .setColor(0x5865F2);

        return message.channel.send({ embeds: [embed] });
    }

    if (content.startsWith('$مزاد')) {
        if (!allowedAuctionChannels.includes(message.channel.id)) {
            return message.reply('❌ لا يمكن بدء مزاد في هذا الروم.');
        }

        if (auctions.has(message.channel.id)) {
            return message.reply('❗ مزاد جاري بالفعل في هذا الروم.');
        }

        const args = content.slice(5).split('|').map(s => s.trim());
        const timeStr = args[0];
        const item = args[1];

        if (!timeStr || !item) {
            return message.reply('❗ الصيغة الصحيحة: $مزاد <المدة> | <اسم العنصر>');
        }

        let minutes = parseInt(timeStr);
        if (isNaN(minutes) || minutes <= 0) {
            return message.reply('⏱️ الرجاء تحديد مدة صحيحة (مثال: 5 دقايق).');
        }

        const durationMs = minutes * 60 * 1000;
        let remainingSeconds = minutes * 60;

        try {
            await message.channel.permissionOverwrites.edit(message.guild.roles.everyone, {
                SendMessages: true
            });
        } catch {}

        const imageUrls = [];
        message.attachments.forEach(att => {
            if (att.contentType && att.contentType.startsWith('image/')) {
                imageUrls.push(att.url);
            }
        });

        const auction = { item };
        auctions.set(message.channel.id, auction);

        const embed = new EmbedBuilder()
            .setTitle('🔨 مزاد جديد!')
            .addFields(
                { name: '🧾 العنصر', value: item },
                { name: '⏳ الوقت المتبقي', value: formatTime(remainingSeconds) },
                { name: '📢 طريقة المزايدة', value: 'اكتب رقمك فقط في هذا الروم' }
            )
            .setColor(0x00AE86);

        if (imageUrls.length > 0) embed.setImage(imageUrls[0]);

        const auctionMsg = await message.channel.send({ embeds: [embed] });
        try { await message.delete(); } catch {}

        const countdownInterval = setInterval(async () => {
            remainingSeconds--;
            if (remainingSeconds <= 0) return;

            const updatedEmbed = EmbedBuilder.from(embed).setFields(
                { name: '🧾 العنصر', value: item },
                { name: '⏳ الوقت المتبقي', value: formatTime(remainingSeconds) },
                { name: '📢 طريقة المزايدة', value: 'اكتب رقمك فقط في هذا الروم' }
            );

            try {
                await auctionMsg.edit({ embeds: [updatedEmbed] });
            } catch {}
        }, 1000);

        setTimeout(async () => {
            clearInterval(countdownInterval);

            try {
                await message.channel.permissionOverwrites.edit(message.guild.roles.everyone, {
                    SendMessages: false
                });
            } catch {}

            try {
                const fetched = await message.channel.messages.fetch({ limit: 100 });
                const messagesToDelete = fetched.filter(msg =>
                    !msg.embeds.length || msg.embeds[0]?.title !== '📘 دليل استخدام بوت المزاد'
                );
                await message.channel.bulkDelete(messagesToDelete, true);

                const endMsg = await message.channel.send('🔒 المزاد انتهى. تم قفل الروم وحذف الرسائل.');
                setTimeout(() => {
                    endMsg.delete().catch(() => {});
                }, 15000);
            } catch {
                await message.channel.send('⚠️ فشل حذف بعض الرسائل.');
            }

            auctions.delete(message.channel.id);
        }, durationMs);
    }
});

client.login(adc2783cd12213853ec0beb156dd96c89a66fed8fb35ecdaa1660dcbfb1abf52);
