import logging
import sqlite3
import datetime
from telegram import Update, ReplyKeyboardMarkup, KeyboardButton, InlineKeyboardMarkup, InlineKeyboardButton, InputMediaPhoto, InputMediaVideo
from telegram.ext import Application, CommandHandler, MessageHandler, ContextTypes, filters, CallbackQueryHandler

# Enable logging
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

# Configuration
TOKEN = "7915140930:AAGMZY1MkNsarvZLFRghBsxedUAxXxzEuT8"  # Replace with your bot token
ADMIN_ID = 6260262910  # Replace with your admin ID
DB_NAME = "file_bot.db"  # Database name
UPLOAD_TYPES = ["video", "photo"]  # Supported file types
NUM_FILES_TO_INITIALIZE = 10000000  # Number of files to initialize

# Initialize database
def create_tables():
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    try:
        c.execute('''CREATE TABLE IF NOT EXISTS files
                     (id INTEGER PRIMARY KEY AUTOINCREMENT,
                      file_id TEXT NOT NULL,
                      file_type TEXT NOT NULL,
                      title TEXT,
                      uploaded_at DATETIME DEFAULT CURRENT_TIMESTAMP)''')
        c.execute('''CREATE TABLE IF NOT EXISTS users
                     (user_id INTEGER PRIMARY KEY,
                      watched_count INTEGER DEFAULT 0,
                      last_watched_date DATE,
                      last_video_upload_date DATE)''')
        conn.commit()
        logger.info("Tables created successfully.")
    except sqlite3.Error as e:
        logger.error(f"Error creating tables: {e}")
    finally:
        if conn:
            conn.close()

create_tables()

# Function to initialize the database with dummy data
def initialize_database(num_files):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    try:
        for i in range(num_files):
            file_id = f"dummy_file_id_{i}"
            file_type = "video" if i % 2 == 0 else "photo"  # Alternate between video and photo
            title = f"Dummy File {i}"
            c.execute("INSERT INTO files (file_id, file_type, title) VALUES (?, ?, ?)",
                      (file_id, file_type, title))
        conn.commit()
        logger.info(f"Initialized database with {num_files} dummy files.")
    except sqlite3.Error as e:
        logger.error(f"Error initializing database: {e}")
    finally:
        if conn:
            conn.close()

# Call the initialization function (only if you want to initialize/re-initialize the DB)
# initialize_database(NUM_FILES_TO_INITIALIZE)  # Comment out after first run


# Keyboards
ADMIN_KEYBOARD = ReplyKeyboardMarkup(
    [[KeyboardButton("📤 Upload Terabox Videos"), KeyboardButton("📝 List Terabox Videos")],
     [KeyboardButton("🚫 Delete File")]],
    resize_keyboard=True
)

USER_KEYBOARD = ReplyKeyboardMarkup(
    [[KeyboardButton("🎬 Premium Watch"), KeyboardButton("Get Video 🍒")]],
    resize_keyboard=True
)

# Helper functions
def get_user_view_count(user_id):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    try:
        c.execute("SELECT watched_count, last_watched_date, last_video_upload_date FROM users WHERE user_id = ?",
                  (user_id,))
        result = c.fetchone()
    except sqlite3.Error as e:
        logger.error(f"Error getting user view count for user {user_id}: {e}")
        return 0, None, None
    finally:
        if conn:
            conn.close()
    if result:
        return result[0], result[1], result[2]
    return 0, None, None


def update_user_view_count(user_id, count, date, upload_date):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    try:
        c.execute(
            "INSERT OR REPLACE INTO users (user_id, watched_count, last_watched_date, last_video_upload_date) VALUES (?, ?, ?, ?)",
            (user_id, count, date, upload_date))
        conn.commit()
    except sqlite3.Error as e:
        logger.error(f"Error updating user view count for user {user_id}: {e}")
    finally:
        if conn:
            conn.close()


def get_last_video_upload_date():
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    try:
        c.execute("SELECT MAX(uploaded_at) FROM files")
        result = c.fetchone()
        if result and result[0]:
            return datetime.datetime.fromisoformat(result[0]).date()
        return None
    except sqlite3.Error as e:
        logger.error(f"Error getting last video upload date: {e}")
        return None
    finally:
        if conn:
            conn.close()


# Handlers
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    user = update.effective_user
    if user.id == ADMIN_ID:
        await update.message.reply_text("👑 Admin Panel", reply_markup=ADMIN_KEYBOARD)
    else:
        await update.message.reply_text(
            f"""Hey {user.first_name} {user.last_name or ''} 👑 \n
            Are you ready for some fun ? 💦
            \nPlease click the  button to get started..""",

            reply_markup=USER_KEYBOARD)


# File Upload Handler (Admin Only)
async def handle_file_upload(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if update.effective_user.id != ADMIN_ID:
        return

    message = update.message
    file = None
    file_type = None

    if message.video:
        file = message.video
        file_type = "video"
    elif message.photo:
        file = message.photo[-1]  # Get the largest resolution photo
        file_type = "photo"
    else:
        
        await update.message.reply_text("Invalid file type. Send a video or photo.")
        return
# Get caption or use empty string
    caption = message.caption or ""

    try:
        conn = sqlite3.connect(DB_NAME)
        c = conn.cursor()
        c.execute("INSERT INTO files (file_id, file_type, title, uploaded_at) VALUES (?, ?, ?, ?)",
                  (file.file_id, file_type, caption, datetime.datetime.now().isoformat()))
        conn.commit()

        # Reset user watch counts for all users
        c.execute("SELECT user_id FROM users")
        users = c.fetchall()
        today = datetime.date.today().isoformat()
        for user in users:
            update_user_view_count(user[0], 0, today, today)

        await update.message.reply_text("✅ Video uploaded successfully!.", reply_markup=ADMIN_KEYBOARD)  # Notify admin

    except sqlite3.Error as e:
        logger.error(f"Error uploading file: {e}")
        await update.message.reply_text(f"Error uploading file: {e}")
        return
    finally:
        if conn:
            conn.close()


# List Files (Admin and Users)
async def list_files(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    try:
        conn = sqlite3.connect(DB_NAME)
        c = conn.cursor()
        c.execute("SELECT id, title, file_type, uploaded_at FROM files")
        files = c.fetchall()

        if not files:
            await update.message.reply_text("No files available.")
            return

        message = "Uploaded Files:\n" + "\n".join(
            [f"ID: {f[0]}, Title: {f[1]}, Type: {f[2]}, Date: {f[3]}" for f in files])
        await update.message.reply_text(message)

    except sqlite3.Error as e:
        logger.error(f"Error listing files: {e}")
        await update.message.reply_text(f"Error listing files: {e}")
        return
    finally:
        if conn:
            conn.close()


# View Files (Users)

async def view_content(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    user_id = update.effective_user.id
    current_date = datetime.date.today().isoformat()
    watched_count, last_watched_date, last_video_upload_date = get_user_view_count(user_id)

    # Daily limit reset:
    if last_watched_date != current_date:
        watched_count = 0

    # File upload reset:
    last_upload = get_last_video_upload_date()  # Get latest Upload date
    if last_upload and last_video_upload_date != last_upload.isoformat():
        watched_count = 0
        update_user_view_count(user_id, 0, current_date, last_upload.isoformat())
        last_video_upload_date = last_upload.isoformat()

    if watched_count is None:
        watched_count = 0  # Initialize if none.

    # Adjust limit based on button pressed
    if update.message.text == "🎬 Premium Watch":
        limit = 100
    elif update.message.text == "Get Video 🍒":
        limit = 3
    else:
        limit = 2  # Default limit (shouldn't happen)

    if watched_count < limit:

        try:
            conn = sqlite3.connect(DB_NAME)
            c = conn.cursor()
            c.execute("SELECT id, file_id, file_type, title FROM files ORDER BY RANDOM() LIMIT 1")
            file_data = c.fetchone()  # Get the file data

            if not file_data:
                await update.message.reply_text("No available.")
                return

            file_id, file_hash, file_type, title = file_data

            try:
                if file_type == "video":
                    await context.bot.send_video(chat_id=update.effective_chat.id, video=file_hash) # Remove caption=title
                elif file_type == "photo":
                    await context.bot.send_photo(chat_id=update.effective_chat.id, photo=file_hash) # Remove caption=title
                else:
                    await update.message.reply_text(f"Unsupported file type: {file_type}")
            except Exception as e:  # Catching broad exceptions as file_id might be invalid
                logger.error(f"Failed to send file: {e}")
                await update.message.reply_text(f"Failed to send file {title}: {e}")

            watched_count += 1  # Now increment after sending and checking there are no errors

            update_user_view_count(user_id, watched_count, current_date, last_video_upload_date)  # Update with correct current_date
            remaining = limit - watched_count
            await update.message.reply_text(f"You can only access {watched_count}/{limit} Premium videos daily.")


        except sqlite3.Error as e:
            logger.error(f"Error fetching files for viewing: {e}")
            await update.message.reply_text(f"Error fetching files for viewing: {e}")
            return
        finally:
            if conn:
                conn.close()

    else:
        await update.message.reply_text(f"You can only access {limit} free videos daily 🔥.")


# Delete Files (Admin Only)
async def handle_delete_menu(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if update.effective_user.id != ADMIN_ID:
        await update.message.reply_text("⛔ Admin access required.")
        return

    try:
        conn = sqlite3.connect(DB_NAME)
        c = conn.cursor()
        c.execute("SELECT id, title FROM files")
        files = c.fetchall()

        if not files:
            await update.message.reply_text("No files to delete.", reply_markup=ADMIN_KEYBOARD)
            return

        keyboard = []
        for file in files:
            keyboard.append([InlineKeyboardButton(f"Delete {file[1]} (ID: {file[0]})", callback_data=f"delete_{file[0]}")])
        keyboard.append([InlineKeyboardButton("💣 Delete All Files", callback_data="delete_all")])

        await update.message.reply_text(
            "Select a file to delete:",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )

    except sqlite3.Error as e:
        logger.error(f"Error listing files for deletion: {e}")
        await update.message.reply_text(f"Error listing files for deletion: {e}")
        return
    finally:
        if conn:
            conn.close()


async def handle_delete_confirmation(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    query = update.callback_query
    await query.answer()

    try:
        conn = sqlite3.connect(DB_NAME)
        c = conn.cursor()

        if query.data == "delete_all":
            # Ask for confirmation
            confirm_keyboard = InlineKeyboardMarkup([
                [InlineKeyboardButton("✅ Confirm", callback_data="confirm_delete_all"),
                 InlineKeyboardButton("❌ Cancel", callback_data="cancel_delete")]
            ])
            await query.edit_message_text(
                "Are you sure you want to delete ALL files? This cannot be undone.",
                reply_markup=confirm_keyboard
            )

        elif query.data.startswith("delete_"):
            file_id = query.data.split("_")[1]
            c.execute("DELETE FROM files WHERE id = ?", (file_id,))
            conn.commit()
            await query.edit_message_text(f"File {file_id} deleted.")


        elif query.data == "confirm_delete_all":
            c.execute("DELETE FROM files")
            conn.commit()
            await query.edit_message_text("All files have been deleted.")


        elif query.data == "cancel_delete":
            await query.edit_message_text("Deletion cancelled.")

        await handle_delete_menu(update, context)  # refresh delete menu


    except sqlite3.Error as e:
        logger.error(f"Error during deletion: {e}")
        await query.edit_message_text(f"Error during deletion: {e}")

    finally:
        if conn:
            conn.close()


def main():
    application = Application.builder().token(TOKEN).build()

    application.add_handler(CommandHandler("start", start))
    application.add_handler(
        MessageHandler(filters.ChatType.PRIVATE & filters.User(ADMIN_ID) & (filters.VIDEO | filters.PHOTO),
                       handle_file_upload))  # Admin file upload
    application.add_handler(
        MessageHandler(filters.Regex(r"📤 Upload Terabox Videos") & filters.User(ADMIN_ID),
                       lambda u, c: u.message.reply_text("Send a file (photo or video) with title.")))
    application.add_handler(MessageHandler(filters.Regex(r"📝 List Terabox Videos"), list_files))  # List files (Admin & Users)

    #The view_content function will handle bot "View Terabox videos" and "View files"
    application.add_handler(MessageHandler(filters.Regex(r"🎬 Premium Watch"), view_content))
    application.add_handler(MessageHandler(filters.Regex(r"Get Video 🍒"), view_content))  # View files (Users)

    # Add delete handlers
    application.add_handler(MessageHandler(filters.Regex(r"🚫 Delete File") & filters.User(ADMIN_ID), handle_delete_menu))
    application.add_handler(CallbackQueryHandler(handle_delete_confirmation, pattern=r"^delete_|confirm_delete_all|cancel_delete"))

    application.run_polling()


if __name__ == "__main__":
    main()
