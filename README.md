# messenger-automation
automaticaly sends messages to messenger groupchats
import os
import time
import json
import ollama
from playwright.sync_api import sync_playwright

# ============================================
# 📄 LOAD CONFIG
# ============================================
def load_config():
    try:
        with open("config.json", "r", encoding="utf-8") as f:
            return json.load(f)
    except Exception as e:
        print(f"❌ Failed to load config.json: {e}")
        return None

# ============================================
# 📚 LOAD DATABASE
# ============================================
def load_database():
    try:
        with open("database.json", "r", encoding="utf-8") as f:
            return json.load(f)
    except Exception as e:
        print(f"❌ Failed to load database.json: {e}")
        return None

# ============================================
# ❌ AUTO CLOSE POPUPS (retained)
# ============================================
def auto_close_popups(page):
    try:
        popup_button = page.locator("div[role='button'][aria-label='Close']")
        if popup_button.is_visible(timeout=500):
            popup_button.click()
            print("❌ Generic popup closed.")

        decline_btn = page.locator(
            "div[aria-label='Decline'], div[aria-label='Ignore'], "
            "button[aria-label='Decline'], button[aria-label='Ignore']"
        )
        if decline_btn.is_visible(timeout=500):
            decline_btn.click()
            print("🚫 Incoming call declined.")

        close_btn = page.locator(
            "div[aria-label='Close'], button[aria-label='Close']"
        )
        if close_btn.is_visible(timeout=500):
            close_btn.click()
            print("❌ Popup closed.")

        overlay = page.locator("div.__fb-light-mode[style*='pointer-events']")
        if overlay.is_visible(timeout=500):
            page.keyboard.press("Escape")
            print("🛑 Overlay dismissed.")
    except:
        pass

# ============================================
# 👀 MONITOR INBOX FOR UNREAD MESSAGES
# ============================================
def get_unread_chats(page):
    try:
        # Look for unread message indicators
        unread = page.locator("div[aria-label*='unread']")
        count = unread.count()
        print(f"📬 Found {count} unread chats.")
        return count
    except Exception as e:
        print(f"❌ Error checking unread: {e}")
        return 0

# ============================================
# 📖 READ LATEST MESSAGE
# ============================================
def read_latest_message(page):
    try:
        # Get all messages in current chat
        messages = page.locator("div[dir='auto']")
        count = messages.count()
        
        if count > 0:
            latest = messages.nth(count - 1).inner_text()
            print(f"📩 Latest message: {latest}")
            return latest
        return None
    except Exception as e:
        print(f"❌ Error reading message: {e}")
        return None

# ============================================
# 🧠 ASK MISTRAL
# ============================================
def ask_mistral(message, database):
    try:
        # Build context from database
        qa_context = json.dumps(database["qa"], ensure_ascii=False)
        
        prompt = f"""
        Ikaw ay isang Real Estate Customer Service Representative.
        
        Narito ang aming Q&A database:
        {qa_context}
        
        Ang tanong ng customer: "{message}"
        
        Sagutin ang tanong gamit ang database.
        Kung wala sa database, sabihing "Sandali lang po, 
        iko-coordinate ko po kayo sa aming agent."
        
        Isagot lamang ang sagot — walang extra explanation.
        """
        
        response = ollama.chat(
            model='mistral',
            messages=[{
                'role': 'user',
                'content': prompt
            }]
        )
        
        answer = response['message']['content']
        print(f"🧠 Mistral answer: {answer}")
        return answer
        
    except Exception as e:
        print(f"❌ Mistral error: {e}")
        return None

# ============================================
# 📎 GET ATTACHMENT FROM DATABASE
# ============================================
def get_attachment(message, database, folder_path):
    try:
        message_lower = message.lower()
        
        for qa in database["qa"]:
            keywords = qa.get("keywords", [])
            for keyword in keywords:
                if keyword.lower() in message_lower:
                    attachment = qa.get("attachment")
                    if attachment:
                        file_path = os.path.join(folder_path, attachment)
                        if os.path.exists(file_path):
                            print(f"📎 Found attachment: {attachment}")
                            return [file_path]
        
        print("⚠️ No specific attachment found.")
        return []
        
    except Exception as e:
        print(f"❌ Error getting attachment: {e}")
        return []

# ============================================
# ✅ AI QUALITY CHECK
# ============================================
def ai_quality_check(original_message, answer):
    try:
        prompt = f"""
        Customer asked: "{original_message}"
        Proposed answer: "{answer}"
        
        Is this answer relevant and appropriate 
        for a Real Estate inquiry? 
        Reply ONLY with YES or NO.
        """
        
        response = ollama.chat(
            model='mistral',
            messages=[{
                'role': 'user',
                'content': prompt
            }]
        )
        
        result = response['message']['content'].strip().upper()
        print(f"✅ AI Quality Check: {result}")
        return "YES" in result
        
    except Exception as e:
        print(f"❌ Quality check error: {e}")
        return True  # Default to send if check fails

# ============================================
# 📤 SEND MESSAGE WITH ATTACHMENTS (retained)
# ============================================
def send_reply(page, message, files=[]):
    try:
        page.wait_for_timeout(2000)
        page.wait_for_selector("div[aria-label='Message']", timeout=30000)
        
        auto_close_popups(page)
        
        # Type message
        input_box = page.locator("div[aria-label='Message']")
        input_box.click()
        input_box.fill(message)
        print("✅ Reply typed.")
        
        # Send attachments if any
        if files:
            try:
                file_input = page.locator("input[type='file']")
                file_input.set_input_files(files)
                print(f"📎 Uploaded {len(files)} files.")
            except Exception as e:
                print(f"❌ Failed to upload files: {e}")
        
        # Wait for uploads
        for _ in range(3):
            auto_close_popups(page)
            time.sleep(2)
        
        # Send
        page.keyboard.press("Enter")
        print("✅ Reply sent!")
        
        # Wait after sending
        for _ in range(5):
            auto_close_popups(page)
            time.sleep(2)
            
    except Exception as e:
        print(f"❌ Error sending reply: {e}")

# ============================================
# 🚀 MAIN
# ============================================
def main():
    config = load_config()
    if not config:
        return
    
    database = load_database()
    if not database:
        return
    
    folder_path = config["folder_path"]
    check_interval = config.get("check_interval", 5)
    
    processed_messages = set()  # Track already replied messages
    
    with sync_playwright() as p:
        print("🌐 Launching Edge...")
        browser = p.chromium.launch_persistent_context(
            user_data_dir=config["edge_profile"],
            headless=False,
            channel="msedge"
        )
        
        page = browser.pages[0] if browser.pages else browser.new_page()
        page.goto("https://www.messenger.com")
        print("📨 Messenger opened.")
        
        page.wait_for_selector('div[role="grid"]', timeout=60000)
        print("✅ Inbox loaded. Monitoring for messages...")
        
        while True:
            try:
                auto_close_popups(page)
                
                # Check for unread chats
                unread_count = get_unread_chats(page)
                
                if unread_count > 0:
                    # Click first unread chat
                    unread_chat = page.locator(
                        "div[aria-label*='unread']"
                    ).first
                    unread_chat.click()
                    print("📬 Opened unread chat.")
                    
                    time.sleep(2)
                    auto_close_popups(page)
                    
                    # Read latest message
                    customer_message = read_latest_message(page)
                    
                    if customer_message and customer_message not in processed_messages:
                        print(f"📩 Processing: {customer_message}")
                        
                        # Ask Mistral
                        answer = ask_mistral(customer_message, database)
                        
                        if answer:
                            # Quality check
                            is_good = ai_quality_check(customer_message, answer)
                            
                            if is_good:
                                # Get attachment
                                files = get_attachment(
                                    customer_message, 
                                    database, 
                                    folder_path
                                )
                                
                                # Send reply
                                send_reply(page, answer, files)
                                
                                # Mark as processed
                                processed_messages.add(customer_message)
                                print("✅ Message processed!")
                            else:
                                print("⚠️ Quality check failed — skipping.")
                
                print(f"⏳ Waiting {check_interval}s before next check...")
                time.sleep(check_interval)
                
            except Exception as e:
                print(f"❌ Main loop error: {e}")
                time.sleep(check_interval)

if __name__ == "__main__":
    main()
