Swapper

import os
import asyncio
import requests
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, MessageHandler, filters, ContextTypes
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import TimeoutException, NoSuchElementException
import sqlite3
import datetime
import random
import time
import logging
import threading
import json

# Database Setup
def init_db():
    conn = sqlite3.connect('swap_bot.db')
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS users
                 (user_id TEXT PRIMARY KEY, username TEXT, credits INT DEFAULT 0, 
                  target_session TEXT, main_session TEXT, swap_status TEXT, 
                  is_running BOOLEAN DEFAULT 0)''')
    conn.commit()
    conn.close()

# Configuration
TOKEN = "8242400375:AAFtSKjsW4o9qx1LotrCB5HQycznLzUoB-I"
ADMIN = "r1yansh"  # Remove @ from admin username
ANIME_GIF = "https://media.giphy.com/media/v1.Y2lkPTc5M2Y4Njg1YzM4NzVhNjQ3YzM5ZDU4ZDU3ZjI0ZDU5ZDU4ZDU3ZjI0ZDU5ZDU4ZDU3ZjI0/giphy.gif"

# Global dictionary to track running swaps
running_swaps = {}

# Free Proxy Sources
FREE_PROXY_APIS = [
    "https://api.proxyscrape.com/v2/?request=get&protocol=http&timeout=10000&country=all&ssl=all&anonymity=all",
    "https://raw.githubusercontent.com/TheSpeedX/PROXY-List/master/http.txt",
    "https://raw.githubusercontent.com/clarketm/proxy-list/master/proxy-list-raw.txt",
    "https://raw.githubusercontent.com/sunny9577/proxy-scraper/master/proxies.txt",
    "https://raw.githubusercontent.com/ShiftyTR/Proxy-List/master/http.txt"
]

# Fetch free proxies
def fetch_free_proxies():
    proxies = []
    for api in FREE_PROXY_APIS:
        try:
            response = requests.get(api, timeout=10)
            if response.status_code == 200:
                proxy_list = response.text.strip().split('\n')
                for proxy in proxy_list:
                    if ':' in proxy and len(proxy.split(':')) == 2:
                        proxies.append(proxy.strip())
        except Exception as e:
            logging.error(f"Failed to fetch from {api}: {e}")
            continue
    
    # Remove duplicates and return
    return list(set(proxies))

# Test proxy functionality
def test_proxy(proxy):
    try:
        proxy_dict = {
            'http': f'http://{proxy}',
            'https': f'http://{proxy}'
        }
        response = requests.get('http://httpbin.org/ip', proxies=proxy_dict, timeout=5)
        return response.status_code == 200
    except:
        return False

# Get working proxy
def get_working_proxy():
    proxies = fetch_free_proxies()
    random.shuffle(proxies)
    
    for proxy in proxies[:10]:  # Test first 10 proxies
        if test_proxy(proxy):
            return proxy
    
    return None  # No working proxy found

# Cache for proxies
proxy_cache = []
last_proxy_fetch = 0

def get_proxy():
    global proxy_cache, last_proxy_fetch
    
    current_time = time.time()
    
    # Refresh proxy cache every 30 minutes
    if current_time - last_proxy_fetch > 1800 or not proxy_cache:
        print("Fetching fresh proxies...")
        proxy_cache = fetch_free_proxies()
        last_proxy_fetch = current_time
    
    if proxy_cache:
        return random.choice(proxy_cache)
    
    return None

# Selenium Setup for Instagram with better proxy handling
def create_driver(proxy=None):
    chrome_options = Options()
    chrome_options.add_argument('--headless')
    chrome_options.add_argument('--no-sandbox')
    chrome_options.add_argument('--disable-dev-shm-usage')
    chrome_options.add_argument('--disable-gpu')
    chrome_options.add_argument('--window-size=1920,1080')
    chrome_options.add_argument('--disable-blink-features=AutomationControlled')
    chrome_options.add_experimental_option("excludeSwitches", ["enable-automation"])
    chrome_options.add_experimental_option('useAutomationExtension', False)
    chrome_options.add_argument('--user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36')
    
    if proxy:
        chrome_options.add_argument(f'--proxy-server=http://{proxy}')
        print(f"Using proxy: {proxy}")
    
    try:
        driver = webdriver.Chrome(options=chrome_options)
        driver.execute_script("Object.defineProperty(navigator, 'webdriver', {get: () => undefined})")
        return driver
    except Exception as e:
        logging.error(f"Failed to create driver: {e}")
        return None

def login_instagram(username, password):
    max_attempts = 3
    
    for attempt in range(max_attempts):
        proxy = get_proxy()
        driver = create_driver(proxy)
        
        if not driver:
            continue
        
        try:
            driver.get("https://www.instagram.com/accounts/login/")
            wait = WebDriverWait(driver, 15)
            
            # Accept cookies if present
            try:
                accept_btn = wait.until(EC.element_to_be_clickable((By.XPATH, "//button[contains(text(), 'Accept') or contains(text(), 'Allow')]")))
                accept_btn.click()
                time.sleep(2)
            except TimeoutException:
                pass
            
            # Login
            username_input = wait.until(EC.presence_of_element_located((By.NAME, "username")))
            password_input = driver.find_element(By.NAME, "password")
            
            username_input.clear()
            username_input.send_keys(username)
            time.sleep(1)
            
            password_input.clear()
            password_input.send_keys(password)
            time.sleep(1)
            
            login_btn = driver.find_element(By.XPATH, "//button[@type='submit']")
            login_btn.click()
            
            time.sleep(5)
            
            # Check for various redirect scenarios
            current_url = driver.current_url
            
            if "challenge" in current_url or "two_factor" in current_url or "checkpoint" in current_url:
                logging.error("2FA/Challenge detected")
                driver.quit()
                continue
            
            # Check if login was successful
            if "instagram.com" in current_url and "login" not in current_url:
                print(f"Login successful for {username}")
                return driver
            else:
                driver.quit()
                continue
                
        except Exception as e:
            logging.error(f"Login attempt {attempt + 1} failed: {e}")
            if driver:
                driver.quit()
            continue
    
    return None

# Database helper functions
def get_user_data(user_id):
    conn = sqlite3.connect('swap_bot.db')
    c = conn.cursor()
    c.execute("SELECT * FROM users WHERE user_id=?", (str(user_id),))
    result = c.fetchone()
    conn.close()
    return result

def update_user_data(user_id, **kwargs):
    conn = sqlite3.connect('swap_bot.db')
    c = conn.cursor()
    
    # Build update query dynamically
    set_clause = ", ".join([f"{key}=?" for key in kwargs.keys()])
    values = list(kwargs.values()) + [str(user_id)]
    
    c.execute(f"UPDATE users SET {set_clause} WHERE user_id=?", values)
    conn.commit()
    conn.close()

# Welcome Message
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    date_time = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    
    conn = sqlite3.connect('swap_bot.db')
    c = conn.cursor()
    c.execute("INSERT OR REPLACE INTO users (user_id, username, credits) VALUES (?, ?, ?)", 
              (str(user.id), user.username, 0))
    conn.commit()
    conn.close()
    
    keyboard = get_main_keyboard()
    
    await update.message.reply_animation(
        animation=ANIME_GIF,
        caption=f"🎌 **Instagram Swap Bot** 🎌\n\n"
                f"Created by: @r1yansh\n"
                f"Started on: {date_time}\n"
                f"Credits: 0\n\n"
                f"⚠️ **Warning**: Use at your own risk!",
        reply_markup=keyboard,
        parse_mode='Markdown'
    )

# Keyboard layouts
def get_main_keyboard():
    return InlineKeyboardMarkup([
        [InlineKeyboardButton("👨‍💻 Developer", callback_data="developer"), 
         InlineKeyboardButton("🔄 Swap", callback_data="swap")],
        [InlineKeyboardButton("❓ Help", callback_data="help"), 
         InlineKeyboardButton("📊 Status", callback_data="status")]
    ])

def get_swap_keyboard():
    return InlineKeyboardMarkup([
        [InlineKeyboardButton("🎯 Target", callback_data="target"), 
         InlineKeyboardButton("🏠 Main", callback_data="main")],
        [InlineKeyboardButton("▶️ Run", callback_data="run"), 
         InlineKeyboardButton("⏹️ Stop", callback_data="stop")],
        [InlineKeyboardButton("🔙 Back", callback_data="back")]
    ])

# Button handler
async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    user_id = query.from_user.id
    user_data = get_user_data(user_id)
    
    if query.data == "developer":
        await query.edit_message_text(
            "👨‍💻 **Developer Info**\n\n"
            "Developer: Riyansh\n"
            "Telegram: @r1yansh\n"
            "Instagram: instagram.com/d7nvn\n\n"
            "⚠️ Use this bot responsibly!",
            parse_mode='Markdown'
        )
        
    elif query.data == "help":
        await query.edit_message_text(
            "❓ **Help & Instructions**\n\n"
            "1. Set target account (username:password)\n"
            "2. Set main account (username:password)\n"
            "3. Click Run to start swapping\n\n"
            "Format: username:password\n"
            "Example: myusername:mypassword\n\n"
            "More info: t.me/riyanshbugs",
            parse_mode='Markdown'
        )
        
    elif query.data == "status":
        if user_data:
            status = user_data[5] if user_data[5] else "No swaps yet"
            is_running = "🟢 Running" if user_data[6] else "🔴 Stopped"
            await query.edit_message_text(
                f"📊 **Status**\n\n"
                f"Credits: {user_data[2]}\n"
                f"Status: {is_running}\n"
                f"Last Result: {status}",
                parse_mode='Markdown'
            )
        
    elif query.data == "swap":
        await query.edit_message_text(
            "🔄 **Swap Menu**\n\n"
            "Select an option:",
            reply_markup=get_swap_keyboard(),
            parse_mode='Markdown'
        )
        
    elif query.data == "target":
        await query.edit_message_text(
            "🎯 **Target Account**\n\n"
            "Send target account credentials:\n"
            "Format: username:password\n\n"
            "⚠️ Make sure the account is valid!"
        )
        context.user_data['waiting_for'] = 'target'
        
    elif query.data == "main":
        await query.edit_message_text(
            "🏠 **Main Account**\n\n"
            "Send main account credentials:\n"
            "Format: username:password\n\n"
            "⚠️ This account will be used for swapping!"
        )
        context.user_data['waiting_for'] = 'main'
        
    elif query.data == "run":
        if not user_data or not user_data[3] or not user_data[4]:
            await query.edit_message_text(
                "❌ **Error**\n\n"
                "Please set both target and main accounts first!"
            )
            return
            
        if user_data[6]:  # Already running
            await query.edit_message_text(
                "⚠️ **Already Running**\n\n"
                "Swap is already in progress!"
            )
            return
            
        await query.edit_message_text(
            "▶️ **Starting Swap**\n\n"
            "Initializing swap process...\n"
            "This may take a few minutes."
        )
        
        # Start swap in background
        threading.Thread(target=run_swap_background, args=(user_id, query.message.chat_id)).start()
        
    elif query.data == "stop":
        if user_id in running_swaps:
            running_swaps[user_id] = False
            update_user_data(user_id, is_running=0)
            await query.edit_message_text(
                "⏹️ **Stopping Swap**\n\n"
                "Swap process will stop shortly..."
            )
        else:
            await query.edit_message_text(
                "⚠️ **No Active Swap**\n\n"
                "No swap is currently running!"
            )
            
    elif query.data == "back":
        await query.edit_message_text(
            "🏠 **Main Menu**\n\n"
            "Select an option:",
            reply_markup=get_main_keyboard()
        )

# Message handler for credentials
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    text = update.message.text
    
    if 'waiting_for' not in context.user_data:
        await update.message.reply_text(
            "❓ Use the buttons to navigate the bot!"
        )
        return
    
    if ":" not in text:
        await update.message.reply_text(
            "❌ Invalid format!\n\n"
            "Use: username:password"
        )
        return
    
    username, password = text.split(":", 1)
    waiting_for = context.user_data['waiting_for']
    
    # Test login
    await update.message.reply_text("🔄 Testing login with proxy...")
    
    driver = login_instagram(username, password)
    if driver:
        driver.quit()
        
        if waiting_for == 'target':
            update_user_data(user_id, target_session=text)
            await update.message.reply_text(
                f"✅ **Target Account Set**\n\n"
                f"Username: {username}\n"
                f"Status: Login successful!"
            )
        elif waiting_for == 'main':
            update_user_data(user_id, main_session=text)
            await update.message.reply_text(
                f"✅ **Main Account Set**\n\n"
                f"Username: {username}\n"
                f"Status: Login successful!"
            )
    else:
        await update.message.reply_text(
            f"❌ **Login Failed**\n\n"
            f"Username: {username}\n"
            f"Error: Invalid credentials or 2FA enabled"
        )
    
    del context.user_data['waiting_for']

# Background swap function with better proxy handling
def run_swap_background(user_id, chat_id):
    try:
        running_swaps[user_id] = True
        update_user_data(user_id, is_running=1)
        
        user_data = get_user_data(user_id)
        target_session = user_data[3]
        main_session = user_data[4]
        
        target_username, target_password = target_session.split(":", 1)
        main_username, main_password = main_session.split(":", 1)
        
        print(f"Starting swap for user {user_id}")
        print(f"Target: {target_username}, Main: {main_username}")
        
        # Login to both accounts with proxy rotation
        main_driver = login_instagram(main_username, main_password)
        target_driver = login_instagram(target_username, target_password)
        
        if not main_driver or not target_driver:
            update_user_data(user_id, swap_status="❌ Login failed for one/both accounts", is_running=0)
            if main_driver:
                main_driver.quit()
            if target_driver:
                target_driver.quit()
            return
        
        attempts = 0
        max_attempts = 50
        temp_username = f"temp{random.randint(100000, 999999)}"
        
        try:
            # Step 1: Change main account username to temp
            print("Step 1: Changing main account to temp username")
            main_driver.get("https://www.instagram.com/accounts/edit/")
            time.sleep(3)
            
            wait = WebDriverWait(main_driver, 10)
            username_field = wait.until(EC.presence_of_element_located((By.NAME, "username")))
            username_field.clear()
            username_field.send_keys(temp_username)
            
            submit_btn = main_driver.find_element(By.XPATH, "//button[@type='submit']")
            submit_btn.click()
            time.sleep(5)
            
            # Step 2: Try to claim original username with target account
            print("Step 2: Attempting to claim username with target account")
            
            while running_swaps.get(user_id, False) and attempts < max_attempts:
                try:
                    # Refresh page and try again
                    target_driver.get("https://www.instagram.com/accounts/edit/")
                    time.sleep(3)
                    
                    wait = WebDriverWait(target_driver, 10)
                    username_field = wait.until(EC.presence_of_element_located((By.NAME, "username")))
                    username_field.clear()
                    username_field.send_keys(main_username)
                    
                    submit_btn = target_driver.find_element(By.XPATH, "//button[@type='submit']")
                    submit_btn.click()
                    time.sleep(5)
                    
                    # Check for success/error messages
                    page_source = target_driver.page_source
                    
                    if "username isn't available" in page_source.lower() or "available" in page_source.lower():
                        attempts += 1
                        print(f"Attempt {attempts}: Username not available, retrying...")
                        update_user_data(user_id, swap_status=f"🔄 Attempt {attempts}: Username not available")
                        
                        # Use random delay to avoid detection
                        delay = random.randint(30, 120)
                        time.sleep(delay)
                        continue
                    
                    # Check if we're still on edit page (success)
                    if "accounts/edit" in target_driver.current_url:
                        # Verify the username changed
                        current_username = target_driver.find_element(By.NAME, "username").get_attribute("value")
                        if current_username == main_username:
                            update_user_data(user_id, 
                                           swap_status=f"✅ SUCCESS! Username swapped in {attempts} attempts",
                                           is_running=0)
                            print(f"SUCCESS: Username swapped in {attempts} attempts")
                            break
                    
                    attempts += 1
                    
                except Exception as e:
                    attempts += 1
                    print(f"Attempt {attempts} error: {e}")
                    update_user_data(user_id, swap_status=f"🔄 Attempt {attempts}: Error occurred")
                    
                    # Random delay between attempts
                    delay = random.randint(60, 180)
                    time.sleep(delay)
                    
                    if attempts >= max_attempts:
                        break
            
            if attempts >= max_attempts:
                update_user_data(user_id, swap_status=f"❌ Failed after {max_attempts} attempts", is_running=0)
                        
        except Exception as e:
            update_user_data(user_id, swap_status=f"❌ Critical error: {str(e)}", is_running=0)
            print(f"Critical error: {e}")
            
        finally:
            print("Cleaning up drivers...")
            if main_driver:
                main_driver.quit()
            if target_driver:
                target_driver.quit()
            running_swaps[user_id] = False
            update_user_data(user_id, is_running=0)
            
    except Exception as e:
        update_user_data(user_id, swap_status=f"❌ System error: {str(e)}", is_running=0)
        running_swaps[user_id] = False
        print(f"System error: {e}")

# Admin commands
async def credit_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.username != ADMIN:
        await update.message.reply_text("❌ Admin only command!")
        return
    
    try:
        args = context.args
        if len(args) != 2:
            await update.message.reply_text("Usage: /credit @username amount")
            return
            
        target_user = args[0].replace("@", "")
        amount = int(args[1])
        
        conn = sqlite3.connect('swap_bot.db')
        c = conn.cursor()
        c.execute("SELECT credits FROM users WHERE username=?", (target_user,))
        result = c.fetchone()
        
        if result:
            new_credits = max(0, result[0] + amount)
            c.execute("UPDATE users SET credits=? WHERE username=?", (new_credits, target_user))
            conn.commit()
            
            action = "Added" if amount > 0 else "Removed"
            await update.message.reply_text(
                f"✅ {action} {abs(amount)} credits for @{target_user}\n"
                f"New balance: {new_credits}"
            )
        else:
            await update.message.reply_text(f"❌ User @{target_user} not found!")
            
        conn.close()
        
    except Exception as e:
        await update.message.reply_text(f"❌ Error: {str(e)}")

# Main function
def main():
    logging.basicConfig(
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        level=logging.INFO
    )
    
    init_db()
    
    # Initialize proxy cache
    print("🔄 Initializing proxy cache...")
    proxy_cache = fetch_free_proxies()
    print(f"✅ Loaded {len(proxy_cache)} proxies")
    
    app = Application.builder().token(TOKEN).build()
    
    # Add handlers
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CallbackQueryHandler(button_handler))
    app.add_handler(CommandHandler("credit", credit_command))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    
    print("🚀 Bot is starting...")
    app.run_polling()

if __name__ == "__main__":
    main()