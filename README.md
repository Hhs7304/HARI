from telegram import Update, InputFile
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters, ConversationHandler, CallbackContext

# Define conversation states
ENTER_FILENAME, RECEIVE_DOCUMENT, ADD_TAG, ADD_DESCRIPTION, SHARE_DOCUMENT, SET_REMINDER, \
RECEIVE_REMINDER, VIEW_REMINDERS, DELETE_REMINDER, SET_CATEGORY, VIEW_CATEGORIES, \
ADD_TO_CATEGORY, VIEW_CATEGORY_DOCUMENTS, SETTINGS = range(14)

# Dictionary to store user data
user_data = {}


# Define a function to handle the /start command
def start(update: Update, _: CallbackContext) -> int:
    update.message.reply_text(
        "Welcome to the Document Bot! 🤖\n"
        "I can help you manage files and notes. Use /help to see the available commands."
    )
    return ENTER_FILENAME


# Define a function to handle the user's input filename
def enter_filename(update: Update, _: CallbackContext) -> int:
    user = update.message.from_user
    user_data[user.id] = {
        'filename': update.message.text,
        'document': None,
        'tags': None,
        'description': None,
        'shared_users': [],
        'reminders': [],
        'category': None
    }
    update.message.reply_text("Please send me the document you want to save.")
    return RECEIVE_DOCUMENT


# Define a function to handle document messages
def receive_document(update: Update, _: CallbackContext) -> int:
    user = update.message.from_user
    if update.message.document:
        user_data[user.id]['document'] = update.message.document
        update.message.reply_text(
            "Document saved successfully! 📄\n"
            "Would you like to add tags to this document? If yes, send me the tags, "
            "separated by commas. If not, type 'skip'.")
    else:
        update.message.reply_text("Please send a valid document.")
    return ADD_TAG


# Define a function to handle adding tags to the document
def add_tag(update: Update, _: CallbackContext) -> int:
    tags = update.message.text.strip()
    if tags.lower() == 'skip':
        update.message.reply_text("No tags added to the document.")
    else:
        user = update.message.from_user
        user_data[user.id]['tags'] = [tag.strip() for tag in tags.split(',')]
        update.message.reply_text(
            "Tags added successfully! 🏷️\n"
            "Would you like to add a description to this document? If yes, send me the "
            "description. If not, type 'skip'.")
    return ADD_DESCRIPTION


# Define a function to handle adding a description to the document
def add_description(update: Update, _: CallbackContext) -> int:
    description = update.message.text.strip()
    if description.lower() == 'skip':
        update.message.reply_text("No description added to the document.")
    else:
        user = update.message.from_user
        user_data[user.id]['description'] = description
        update.message.reply_text(
            "Description added successfully! 📝\n"
            "Your document is now saved with all the details!")
    return ENTER_FILENAME


# Define a function to handle document messages
def receive_document(update: Update, _: CallbackContext) -> int:
    user = update.message.from_user
    if update.message.document:
        user_data[user.id]['document'] = update.message.document
        update.message.reply_text("Document saved successfully! 📄\n"
                                  "Would you like to add tags to this document? If yes, send me the tags, "
                                  "separated by commas. If not, type 'skip'.")
    else:
        update.message.reply_text("Please send a valid document or type 'skip' to skip adding a document.")
    return ADD_TAG



# Define a function to handle the /getdoc command
def get_document(update: Update, _: CallbackContext):
    args = update.message.text.split(' ')[1:]
    if len(args) > 0:
        requested_filename = '_'.join(args)
        user = update.message.from_user
        data = user_data.get(user.id)
        if data and data['filename'] == requested_filename and data['document']:
            update.message.reply_document(document=data['document'].file_id,
                                          filename=data['filename'])
            if data['tags']:
                tags_text = ", ".join(data['tags'])
                update.message.reply_text(f"Tags: {tags_text}")
            if data['description']:
                update.message.reply_text(f"Description: {data['description']}")
        else:
            update.message.reply_text("File not found.")
    else:
        update.message.reply_text("Please provide a filename after the command.")


# Define a function to handle text messages and renaming files
def text_handler(update: Update, _: CallbackContext):
    user = update.message.from_user
    data = user_data.get(user.id)
    if data and data['document']:
        data['filename'] = update.message.text
        user_data[user.id] = data
        update.message.reply_text(f"File '{update.message.text}' renamed.")
    else:
        update.message.reply_text(
            "Please first save a document using /start command and upload a document."
        )


# Define a function to handle saving text notes with keywords
def save_note(update: Update, _: CallbackContext):
    args = update.message.text.split(' ')[1:]
    if len(args) >= 2:
        keyword, note = args[0].lower(), ' '.join(args[1:])
        user = update.message.from_user
        user_data.setdefault(user.id, {}).setdefault('notes', {})[keyword] = note
        update.message.reply_text(f"Note '{note}' saved with keyword '{keyword}'.")
    else:
        update.message.reply_text(
            "Please provide a keyword and note text after the command.")


# Define a function to retrieve notes by keywords
def get_note_by_keyword(update: Update, _: CallbackContext):
    args = update.message.text.split(' ')[1:]
    if len(args) > 0:
        requested_keyword = args[0].lower()
        user = update.message.from_user
        notes = user_data.get(user.id, {}).get('notes', {})
        note = notes.get(requested_keyword)
        if note:
            update.message.reply_text(
                f"Note for keyword '{requested_keyword}': {note}")
        else:
            update.message.reply_text(
                f"Note for keyword '{requested_keyword}' not found.")
    else:
        update.message.reply_text("Please provide a keyword after the command.")


# Define a function to handle the /listdocs command
def list_documents(update: Update, _: CallbackContext):
    user = update.message.from_user
    filenames = [
        data['filename'] for data in user_data.values() if data.get('filename')
    ]
    if filenames:
        doc_list_text = "Your stored documents:\n" + "\n".join(filenames)
    else:
        doc_list_text = "No documents found. Use /start command to save documents."
    update.message.reply_text(doc_list_text)


# Define a function to handle the /delete command
def delete_document(update: Update, _: CallbackContext):
    args = update.message.text.split(' ')[1:]
    if len(args) > 0:
        document_name = '_'.join(args)
        user = update.message.from_user
        data = user_data.get(user.id)
        if data and data['filename'] == document_name:
            user_data[user.id] = {}
            update.message.reply_text(
                f"Document '{document_name}' deleted successfully!")
        else:
            update.message.reply_text("Document not found.")
    else:
        update.message.reply_text("Please provide a filename after the command.")


# Define a function to handle the /clear command
def clear_data(update: Update, _: CallbackContext):
    user = update.message.from_user
    user_data[user.id] = {}
    update.message.reply_text("All your data has been cleared successfully!")


# Define a function to handle the /search command
def search_documents(update: Update, _: CallbackContext):
    args = update.message.text.split(' ')[1:]
    if len(args) > 0:
        search_query = ' '.join(args).lower()
        user = update.message.from_user
        documents = [
            data for data in user_data.values()
            if data.get('filename', '').lower().find(search_query) != -1
        ]
        if documents:
            doc_list_text = "Search results for query: " + search_query + "\n"
            doc_list_text += "\n".join([data['filename'] for data in documents])
            update.message.reply_text(doc_list_text)
        else:
            update.message.reply_text(
                "No documents found for the given search query.")
    else:
        update.message.reply_text(
            "Please provide a search query after the command.")


# Define a function to handle adding documents to favorites
def add_to_favorites(update: Update, _: CallbackContext):
    args = update.message.text.split(' ')[1:]
    if len(args) > 0:
        document_name = '_'.join(args)
        user = update.message.from_user
        document = [
            data for data in user_data.values()
            if data.get('filename') == document_name
        ]
        if document:
            user_data.setdefault(user.id, {}).setdefault('favorites',
                                                          []).append(document_name)
            update.message.reply_text(
                f"Document '{document_name}' added to favorites.")
        else:
            update.message.reply_text("Document not found.")
    else:
        update.message.reply_text(
            "Please provide a document name after the command.")


# Define a function to handle viewing favorite documents
def view_favorites(update: Update, _: CallbackContext):
    user = update.message.from_user
    favorites = user_data.get(user.id, {}).get('favorites', [])
    if favorites:
        fav_list_text = "Your favorite documents:\n" + "\n".join(favorites)
    else:
        fav_list_text = "No favorite documents found. Use /addtofavorites command to add documents to favorites."
    update.message.reply_text(fav_list_text)


# Define a function to handle document statistics
def document_statistics(update: Update, _: CallbackContext):
    user = update.message.from_user
    num_documents = len(
        [data for data in user_data.values() if data.get('filename')])
    num_notes = len(user_data.get(user.id, {}).get('notes', {}))
    update.message.reply_text(f"Statistics:\n"
                              f"Total Documents: {num_documents}\n"
                              f"Total Notes: {num_notes}")


# Define a function to handle document sharing
def share_document(update: Update, _: CallbackContext) -> int:
    user = update.message.from_user
    data = user_data.get(user.id)
    if data and data['document']:
        update.message.reply_text(
            "Please provide the username of the user you want to share the document with."
        )
        return SHARE_DOCUMENT
    else:
        update.message.reply_text(
            "Please first save a document using /start command and upload a document."
        )
        return ENTER_FILENAME


# Define a function to handle sharing the document with a user
def share_document_with_user(update: Update, _: CallbackContext) -> int:
    shared_user = update.message.text.strip()
    user = update.message.from_user
    data = user_data.get(user.id)
    if data and data['document']:
        data['shared_users'].append(shared_user)
        user_data[user.id] = data
        update.message.reply_text(
            f"Document shared with user {shared_user} successfully!")
    else:
        update.message.reply_text(
            "Please first save a document using /start command and upload a document."
        )
    return ENTER_FILENAME


# Define a function to handle setting a reminder for a document
def set_reminder(update: Update, _: CallbackContext) -> int:
    user = update.message.from_user
    data = user_data.get(user.id)
    if data and data['document']:
        update.message.reply_text(
            "Please provide the date and time (in format YYYY-MM-DD HH:MM) for the reminder."
        )
        return SET_REMINDER
    else:
        update.message.reply_text(
            "Please first save a document using /start command and upload a document."
        )
        return ENTER_FILENAME


# Define a function to handle receiving the reminder date and time
def receive_reminder(update: Update, _: CallbackContext) -> int:
    reminder_datetime = update.message.text.strip()
    user = update.message.from_user
    data = user_data.get(user.id)
    if data and data['document']:
        data['reminders'].append(reminder_datetime)
        user_data[user.id] = data
        update.message.reply_text(
            f"Reminder set for {reminder_datetime} successfully!")
    else:
        update.message.reply_text(
            "Please first save a document using /start command and upload a document."
        )
    return ENTER_FILENAME


# Define a function to handle viewing all reminders
def view_reminders(update: Update, _: CallbackContext) -> int:
    user = update.message.from_user
    reminders = user_data.get(user.id, {}).get('reminders', [])
    if reminders:
        reminder_list_text = "Your reminders:\n" + "\n".join(reminders)
    else:
        reminder_list_text = "No reminders found. Use /setreminder command to add reminders for documents."
    update.message.reply_text(reminder_list_text)
    return ENTER_FILENAME


# Define a function to handle deleting a reminder
def delete_reminder(update: Update, _: CallbackContext) -> int:
    user = update.message.from_user
    reminders = user_data.get(user.id, {}).get('reminders', [])
    if len(reminders) > 0:
        update.message.reply_text(
            "Please provide the index of the reminder you want to delete (starting from 1)."
        )
        return DELETE_REMINDER
    else:
        update.message.reply_text("No reminders found to delete.")
        return ENTER_FILENAME


# Define a function to handle receiving the index of the reminder to delete
def receive_delete_reminder(update: Update, _: CallbackContext) -> int:
    index = int(update.message.text.strip()) - 1
    user = update.message.from_user
    reminders = user_data.get(user.id, {}).get('reminders', [])
    if 0 <= index < len(reminders):
        del reminders[index]
        user_data[user.id]['reminders'] = reminders
        update.message.reply_text("Reminder deleted successfully!")
    else:
        update.message.reply_text(
            "Invalid reminder index. Please provide a valid index.")
    return ENTER_FILENAME


# Define a function to handle setting a category for a document
def set_category(update: Update, _: CallbackContext) -> int:
    user = update.message.from_user
    data = user_data.get(user.id)
    if data and data['document']:
        update.message.reply_text(
            "Please provide a category name for this document.")
        return SET_CATEGORY
    else:
        update.message.reply_text(
            "Please first save a document using /start command and upload a document."
        )
        return ENTER_FILENAME


# Define a function to handle receiving the category name for the document
def receive_category(update: Update, _: CallbackContext) -> int:
    category_name = update.message.text.strip()
    user = update.message.from_user
    data = user_data.get(user.id)
    if data and data['document']:
        data['category'] = category_name
        user_data[user.id] = data
        update.message.reply_text(
            f"Category '{category_name}' set for the document successfully!")
    else:
        update.message.reply_text(
            "Please first save a document using /start command and upload a document."
        )
    return ENTER_FILENAME


# Define a function to handle viewing all categories
def view_categories(update: Update, _: CallbackContext) -> int:
    user = update.message.from_user
    categories = list(
        set([
            data['category'] for data in user_data.values()
            if data.get('category')
        ]))
    if categories:
        category_list_text = "Your categories:\n" + "\n".join(categories)
    else:
        category_list_text = "No categories found. Use /setcategory command to add categories for documents."
    update.message.reply_text(category_list_text)
    return ENTER_FILENAME


# Define a function to handle adding a document to a category
def add_to_category(update: Update, _: CallbackContext) -> int:
    args = update.message.text.split(' ')[1:]
    if len(args) >= 2:
        category_name = '_'.join(args[:-1])
        document_name = args[-1]
        user = update.message.from_user
        data = user_data.get(user.id)
        if data and data['document'] and data['filename'] == document_name:
            data['category'] = category_name
            user_data[user.id] = data
            update.message.reply_text(
                f"Document '{document_name}' added to category '{category_name}' successfully!"
            )
        else:
            update.message.reply_text(
                "Document not found or you don't have permission to add it to the category."
            )
    else:
        update.message.reply_text(
            "Please provide a category name and document name after the command."
        )
    return ENTER_FILENAME


# Define a function to handle viewing all documents in a category
def view_category_documents(update: Update, _: CallbackContext) -> int:
    args = update.message.text.split(' ')[1:]
    if len(args) > 0:
        category_name = '_'.join(args)
        user = update.message.from_user
        documents = [
            data['filename'] for data in user_data.values()
            if data.get('category') == category_name
        ]
        if documents:
            doc_list_text = f"Documents in category '{category_name}':\n" + "\n".join(
                documents)
        else:
            doc_list_text = f"No documents found in category '{category_name}'."
        update.message.reply_text(doc_list_text)
    else:
        update.message.reply_text(
            "Please provide a category name after the command.")
    return ENTER_FILENAME


# Define a function to handle the /about command
def about_command(update: Update, _: CallbackContext):
    update.message.reply_text(
        "This bot helps you manage documents and notes. You can save documents, add tags and descriptions, set "
        "reminders, categorize documents, add documents to favorites, share documents with other users, and much more! "
        "Use /help to see the available commands.")


# Define a function to handle the /help command
def help_command(update: Update, _: CallbackContext):
    help_text = (
        "Here are the available commands:\n\n"
        "/start - Start the bot and save a document.\n"
        "/getdoc <filename> - Get a saved document by its filename.\n"
        "/listdocs - List all your saved documents.\n"
        "/delete <filename> - Delete a saved document.\n"
        "/clear - Clear all your data.\n"
        "/about - About this bot.\n"
        "/search <query> - Search for documents by name.\n"
        "/addtofavorites <filename> - Add a document to favorites.\n"
        "/viewfavorites - View your favorite documents.\n"
        "/statistics - View statistics about your saved documents and notes.\n"
        "/savenote <keyword> <note> - Save a text note with a keyword.\n"
        "/getnote <keyword> - Get a note by its keyword.\n"
        "/share - Share a document with another user.\n"
        "/setreminder - Set a reminder for a saved document.\n"
        "/viewreminders - View all your reminders.\n"
        "/deletereminder - Delete a reminder.\n"
        "/setcategory - Set a category for a saved document.\n"
        "/viewcategories - View all your categories.\n"
        "/addtocategory <category> <filename> - Add a document to a category.\n"
        "/viewcategorydocuments <category> - View all documents in a category.\n"
        "/help - Show this help message.\n")
    update.message.reply_text(help_text)


# Create an instance of the Updater class and pass in your bot token
updater = Updater(token='6133585030:AAGXUU3Cm2eLJDHduTLF3oIl5-kRBWHg1JE', use_context=True)

# Get the dispatcher to register handlers
dispatcher = updater.dispatcher

# Add command handlers
dispatcher.add_handler(CommandHandler('start', start))
dispatcher.add_handler(CommandHandler('getdoc', get_document))
dispatcher.add_handler(CommandHandler('help', help_command))
dispatcher.add_handler(CommandHandler('listdocs', list_documents))
dispatcher.add_handler(CommandHandler('delete', delete_document))
dispatcher.add_handler(CommandHandler('clear', clear_data))
dispatcher.add_handler(CommandHandler('about', about_command))
dispatcher.add_handler(CommandHandler('search', search_documents))
dispatcher.add_handler(CommandHandler('addtofavorites', add_to_favorites))
dispatcher.add_handler(CommandHandler('viewfavorites', view_favorites))
dispatcher.add_handler(CommandHandler('statistics', document_statistics))
dispatcher.add_handler(CommandHandler('share', share_document))
dispatcher.add_handler(CommandHandler('setreminder', set_reminder))
dispatcher.add_handler(CommandHandler('viewreminders', view_reminders))
dispatcher.add_handler(CommandHandler('deletereminder', delete_reminder))
dispatcher.add_handler(CommandHandler('setcategory', set_category))
dispatcher.add_handler(CommandHandler('viewcategories', view_categories))
dispatcher.add_handler(CommandHandler('addtocategory', add_to_category))
dispatcher.add_handler(
    CommandHandler('viewcategorydocuments', view_category_documents))

# Add message handlers
dispatcher.add_handler(MessageHandler(Filters.text & ~Filters.command,
                                      text_handler))
dispatcher.add_handler(
    MessageHandler(Filters.document.mime_type("text/plain"), save_note))
dispatcher.add_handler(
    MessageHandler(Filters.document.mime_type("application/pdf"),
                   receive_document))

# Add conversation handlers
conv_handler = ConversationHandler(
    entry_points=[CommandHandler('start', start)],
    states={
        ENTER_FILENAME: [MessageHandler(Filters.text, enter_filename)],
        RECEIVE_DOCUMENT: [
            MessageHandler(Filters.document.mime_type("application/pdf"),
                           receive_document)
        ],
        ADD_TAG: [MessageHandler(Filters.text, add_tag)],
        ADD_DESCRIPTION: [MessageHandler(Filters.text, add_description)],
        SHARE_DOCUMENT: [MessageHandler(Filters.text, share_document_with_user)],
        SET_REMINDER: [MessageHandler(Filters.text, receive_reminder)],
        RECEIVE_REMINDER: [MessageHandler(Filters.text, receive_reminder)],
        VIEW_REMINDERS: [MessageHandler(Filters.text, view_reminders)],
        DELETE_REMINDER: [MessageHandler(Filters.text, receive_delete_reminder)],
        SET_CATEGORY: [MessageHandler(Filters.text, receive_category)],
        VIEW_CATEGORIES: [MessageHandler(Filters.text, view_categories)],
        ADD_TO_CATEGORY: [MessageHandler(Filters.text, add_to_category)],
        VIEW_CATEGORY_DOCUMENTS: [
            MessageHandler(Filters.text, view_category_documents)
        ],
        SETTINGS: [MessageHandler(Filters.text, text_handler)],
    },
    fallbacks=[CommandHandler('cancel', start)],
)

dispatcher.add_handler(conv_handler)

# Start the Bot
updater.start_polling()

# Run the bot until the user presses Ctrl-C
updater.idle()
