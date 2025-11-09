# run.py

import telebot
from telebot import types
import re

# ==============================================================================
# 🔴 WARNING: REPLACE WITH YOUR NEW, SECURE TOKEN 🔴
# Your old token is compromised. Get a new one from @BotFather
TOKEN = '8237313309:AAEXzBBsQq4dV1auo9pJN6OcM8SNyjYCgO0'
# ==============================================================================

bot = telebot.TeleBot(TOKEN)

# --- Data Storage (In-Memory Dictionary) ---
# In a real application, use a proper database like SQLite or PostgreSQL.
users_db = {}

# --- User State Management ---
# Tracks what each user is currently doing (e.g., signing up, changing settings).
user_states = {}

# --- Helper Functions ---

def is_valid_email(email):
    """Checks if the email format is valid."""
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return re.match(pattern, email) is not None

def get_main_menu():
    """Creates the main home page menu."""
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    markup.add(
        types.KeyboardButton('👤 Profile'),
        types.KeyboardButton('⚙️ Settings'),
        types.KeyboardButton('🚪 Logout')
    )
    return markup

def get_settings_menu():
    """Creates the settings menu."""
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    markup.add(
        types.KeyboardButton('✏️ Change Name'),
        types.KeyboardButton('✉️ Change Email'),
        types.KeyboardButton('🔙 Back to Home')
    )
    return markup

def get_confirmation_menu():
    """Creates a temporary menu for Yes/No confirmation."""
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    markup.add(
        types.KeyboardButton('✅ Submit'),
        types.KeyboardButton('❌ Cancel')
    )
    return markup

# --- Bot Handlers ---

@bot.message_handler(commands=['start'])
def start_message(message):
    """Handles the /start command."""
    chat_id = message.chat.id
    
    if chat_id in users_db:
        bot.send_message(chat_id, "✨ You are already logged in! Welcome back.", reply_markup=get_main_menu())
        return

    # Start the signup process
    user_states[chat_id] = {'state': 'signing_up', 'data': {}}
    msg = bot.send_message(chat_id, "👋 Welcome! Let's create your account.\n\nFirst, please enter your **full name**:")
    bot.register_next_step_handler(msg, process_signup_name)

def process_signup_name(message):
    """Processes the name during signup."""
    chat_id = message.chat.id
    name = message.text
    
    if chat_id not in user_states or user_states[chat_id]['state'] != 'signing_up':
        start_message(message)
        return
        
    user_states[chat_id]['data']['name'] = name
    msg = bot.send_message(chat_id, "Great! Now, please enter your **email address**:")
    bot.register_next_step_handler(msg, process_signup_email)

def process_signup_email(message):
    """Processes the email during signup."""
    chat_id = message.chat.id
    email = message.text
    
    if chat_id not in user_states or user_states[chat_id]['state'] != 'signing_up':
        start_message(message)
        return

    if not is_valid_email(email):
        msg = bot.send_message(chat_id, "❌ That doesn't look like a valid email. Please try again:")
        bot.register_next_step_handler(msg, process_signup_email)
        return

    user_states[chat_id]['data']['email'] = email
    msg = bot.send_message(chat_id, "Excellent! Finally, please create a **password**:")
    bot.register_next_step_handler(msg, process_signup_password)

def process_signup_password(message):
    """Processes the password and asks for confirmation."""
    chat_id = message.chat.id
    password = message.text

    if chat_id not in user_states or user_states[chat_id]['state'] != 'signing_up':
        start_message(message)
        return

    user_states[chat_id]['data']['password'] = password
    
    # Show confirmation buttons using Reply Keyboard
    user_states[chat_id]['state'] = 'signup_confirmation'
    bot.send_message(chat_id, "✨ Please review your information. Press 'Submit' to create your account or 'Cancel' to abort.", reply_markup=get_confirmation_menu())

@bot.message_handler(func=lambda message: message.text in ['✅ Submit', '❌ Cancel'])
def handle_signup_confirmation(message):
    """Handles the submit or cancel action during signup."""
    chat_id = message.chat.id
    
    # Ensure the user is in the correct state to see these buttons
    if chat_id not in user_states or user_states[chat_id].get('state') != 'signup_confirmation':
        return # Ignore if the user is not in the signup confirmation step

    if message.text == '✅ Submit':
        # Save user data to the "database"
        users_db[chat_id] = user_states[chat_id]['data']
        del user_states[chat_id] # Clear the state
        
        bot.send_message(chat_id, "🎉 Congratulations! Your account has been created successfully.", reply_markup=types.ReplyKeyboardRemove())
        
        # Start the login process
        msg = bot.send_message(chat_id, "🔐 Please log in with your new credentials.\n\nEnter your **email**:")
        user_states[chat_id] = {'state': 'logging_in'}
        bot.register_next_step_handler(msg, process_login_email)

    elif message.text == '❌ Cancel':
        if chat_id in user_states:
            del user_states[chat_id]
        bot.send_message(chat_id, "❌ Registration cancelled. Type /start to try again.", reply_markup=types.ReplyKeyboardRemove())

def process_login_email(message):
    """Processes the email during login."""
    chat_id = message.chat.id
    email = message.text

    if chat_id not in user_states or user_states[chat_id]['state'] != 'logging_in':
        start_message(message)
        return

    user_states[chat_id]['email'] = email
    msg = bot.send_message(chat_id, "Now, please enter your **password**:")
    bot.register_next_step_handler(msg, process_login_password)

def process_login_password(message):
    """Validates login credentials."""
    chat_id = message.chat.id
    password = message.text

    if chat_id not in user_states or user_states[chat_id]['state'] != 'logging_in':
        start_message(message)
        return

    email = user_states[chat_id]['email']
    
    # Find user in the database
    user_found = None
    for uid, user_data in users_db.items():
        if user_data['email'] == email:
            user_found = uid
            break
            
    if user_found and users_db[user_found]['password'] == password:
        # Login successful
        del user_states[chat_id] # Clear state
        bot.send_message(chat_id, "✅ Login Successful! Welcome to your Home Page.", reply_markup=get_main_menu())
    else:
        # Login failed
        bot.send_message(chat_id, "❌ Incorrect email or password. Please try again.\n\nEnter your **email**:")
        # Restart login process
        user_states[chat_id] = {'state': 'logging_in'}
        bot.register_next_step_handler(message, process_login_email)

@bot.message_handler(func=lambda message: True)
def handle_text_messages(message):
    """Handles all text messages for navigation and actions."""
    chat_id = message.chat.id
    text = message.text

    # User must be logged in to access these features
    if chat_id not in users_db:
        bot.send_message(chat_id, "🛑 You need to log in first. Please type /start to begin.")
        return

    # --- Main Menu Actions ---
    if text == '👤 Profile':
        user_data = users_db[chat_id]
        profile_text = "👤 --- Your Profile --- 👤\n\n"
        profile_text += f"🔹 *Name:* {user_data['name']}\n"
        profile_text += f"🔹 *Email:* {user_data['email']}\n"
        bot.send_message(chat_id, profile_text, parse_mode='Markdown', reply_markup=get_main_menu())

    elif text == '⚙️ Settings':
        bot.send_message(chat_id, "⚙️ --- Settings ---\n\nWhat would you like to change?", reply_markup=get_settings_menu())
        user_states[chat_id] = {'state': 'in_settings'}

    elif text == '🚪 Logout':
        if chat_id in users_db:
            del users_db[chat_id]
        if chat_id in user_states:
            del user_states[chat_id]
        bot.send_message(chat_id, "You have been successfully logged out. 👋", reply_markup=types.ReplyKeyboardRemove())

    # --- Settings Menu Actions ---
    elif chat_id in user_states and user_states[chat_id].get('state') == 'in_settings':
        
        if text == '✏️ Change Name':
            msg = bot.send_message(chat_id, "Please enter your new name:")
            bot.register_next_step_handler(msg, process_name_change)

        elif text == '✉️ Change Email':
            msg = bot.send_message(chat_id, "Please enter your new email address:")
            bot.register_next_step_handler(msg, process_email_change)

        elif text == '🔙 Back to Home':
            del user_states[chat_id] # Exit settings state
            bot.send_message(chat_id, "Returning to Home Page... 🏠", reply_markup=get_main_menu())

def process_name_change(message):
    chat_id = message.chat.id
    new_name = message.text
    users_db[chat_id]['name'] = new_name
    del user_states[chat_id] # Exit settings state
    bot.send_message(chat_id, "✅ Your name has been successfully updated!", reply_markup=get_main_menu())

def process_email_change(message):
    chat_id = message.chat.id
    new_email = message.text
    
    if not is_valid_email(new_email):
        msg = bot.send_message(chat_id, "❌ Invalid email format. Please enter a valid email:")
        bot.register_next_step_handler(msg, process_email_change)
        return
        
    users_db[chat_id]['email'] = new_email
    del user_states[chat_id] # Exit settings state
    bot.send_message(chat_id, "✅ Your email has been successfully updated!", reply_markup=get_main_menu())


print("✅ Bot is running and designed to your specifications...")
bot.polling(non_stop=True)
