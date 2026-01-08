import random
import json
import os
import copy
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, ContextTypes

# Bot token
BOT_TOKEN = "7965779663:AAE74eK31YVNUxV497dv1dRiZEOKNS92ts8"

# Game constants
GRID_ROWS = 10
GRID_COLS = 3
STARTING_COINS = 100000
MINE_PROBABILITY = 0.2  # 20% chance of mine in each cell
COIN_REWARD = 5000  # Coins for completing the game

# User data storage (in production, use a database)
user_data = {}

def load_user_data():
    """Load user data from file if exists"""
    global user_data
    if os.path.exists('user_data.json'):
        try:
            with open('user_data.json', 'r', encoding='utf-8') as f:
                loaded_data = json.load(f)
            # Convert string keys to integers and revealed lists back to sets
            user_data = {}
            for user_id_str, data in loaded_data.items():
                try:
                    user_id = int(user_id_str)  # Convert string key to integer
                    if data.get('game_state') and 'revealed' in data['game_state']:
                        if isinstance(data['game_state']['revealed'], list):
                            # Convert list of lists to set of tuples
                            data['game_state']['revealed'] = set(tuple(item) for item in data['game_state']['revealed'])
                    user_data[user_id] = data
                except (ValueError, KeyError, TypeError) as e:
                    print(f"Error loading data for user {user_id_str}: {e}")
                    continue
        except (json.JSONDecodeError, IOError) as e:
            print(f"Error loading user_data.json: {e}")
            user_data = {}

def save_user_data():
    """Save user data to file"""
    try:
        # Convert sets to lists for JSON serialization
        data_to_save = {}
        for user_id, data in user_data.items():
            # Use deep copy to avoid modifying original data
            data_copy = copy.deepcopy(data)
            if data_copy.get('game_state') and 'revealed' in data_copy['game_state']:
                # Convert set of tuples to list of lists
                if isinstance(data_copy['game_state']['revealed'], set):
                    data_copy['game_state']['revealed'] = [list(item) for item in data_copy['game_state']['revealed']]
            # Convert integer keys to strings for JSON (JSON requires string keys)
            data_to_save[str(user_id)] = data_copy
        
        with open('user_data.json', 'w', encoding='utf-8') as f:
            json.dump(data_to_save, f, ensure_ascii=False, indent=2)
    except (IOError, TypeError) as e:
        print(f"Error saving user_data.json: {e}")

def init_user(user_id):
    """Initialize user with starting coins"""
    if user_id not in user_data:
        user_data[user_id] = {
            'coins': STARTING_COINS,
            'game_state': None
        }
        save_user_data()

def generate_grid():
    """Generate a random 10x3 grid with mines"""
    grid = []
    for row in range(GRID_ROWS):
        grid_row = []
        for col in range(GRID_COLS):
            # First and last rows are safe (start and finish)
            if row == 0 or row == GRID_ROWS - 1:
                grid_row.append('safe')
            else:
                # Random chance of mine
                if random.random() < MINE_PROBABILITY:
                    grid_row.append('mine')
                else:
                    grid_row.append('safe')
        grid.append(grid_row)
    
    # Ensure at least one safe path exists (start and finish columns are always safe)
    grid[0] = ['safe', 'safe', 'safe']
    grid[GRID_ROWS - 1] = ['safe', 'safe', 'safe']
    
    return grid

def get_grid_display(grid, current_row, current_col, revealed):
    """Create a visual representation of the grid"""
    display = "💎 DIAMOND GAME 💎\n\n"
    display += f"Row: {current_row + 1}/{GRID_ROWS}\n"
    display += "━━━━━━━━━━━━━━━━\n\n"
    
    for row_idx, row in enumerate(grid):
        row_display = ""
        for col_idx, cell in enumerate(row):
            if row_idx == current_row and col_idx == current_col:
                row_display += "👤 "  # Player position
            elif (row_idx, col_idx) in revealed:
                if cell == 'mine':
                    row_display += "💣 "
                else:
                    row_display += "✅ "
            elif row_idx < current_row:
                # Already passed rows
                if cell == 'mine':
                    row_display += "💣 "
                else:
                    row_display += "✅ "
            else:
                # Unexplored
                row_display += "⬜ "
        display += row_display + "\n"
    
    display += "\n━━━━━━━━━━━━━━━━\n"
    return display

def get_movement_keyboard(current_row, current_col):
    """Create keyboard for movement"""
    keyboard = []
    
    # Can move forward if not at the end
    if current_row < GRID_ROWS - 1:
        row_buttons = []
        # Can move to any column in the next row
        for col in range(GRID_COLS):
            row_buttons.append(InlineKeyboardButton(
                f"⬇️ Col {col + 1}",
                callback_data=f"move_{current_row + 1}_{col}"
            ))
        keyboard.append(row_buttons)
    
    # Show current position info
    keyboard.append([InlineKeyboardButton("📍 My Position", callback_data="position")])
    keyboard.append([InlineKeyboardButton("💰 My Coins", callback_data="coins")])
    keyboard.append([InlineKeyboardButton("🔄 New Game", callback_data="new_game")])
    
    return InlineKeyboardMarkup(keyboard)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle /start command"""
    user_id = update.effective_user.id
    init_user(user_id)
    
    welcome_msg = (
        f"👋 Welcome to Diamond Game!\n\n"
        f"💰 Starting Coins: {STARTING_COINS:,}\n\n"
        f"🎮 How to play:\n"
        f"• Navigate through a {GRID_ROWS}x{GRID_COLS} grid\n"
        f"• Avoid mines (💣) to reach the end\n"
        f"• Complete the game to earn {COIN_REWARD:,} coins!\n\n"
        f"Use /game to start playing!"
    )
    
    await update.message.reply_text(welcome_msg)

async def game(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Start a new game"""
    user_id = update.effective_user.id
    init_user(user_id)
    
    # Generate new grid
    grid = generate_grid()
    
    # Start at row 0, middle column
    current_row = 0
    current_col = 1
    
    # Store game state
    user_data[user_id]['game_state'] = {
        'grid': grid,
        'current_row': current_row,
        'current_col': current_col,
        'revealed': set([(0, 0), (0, 1), (0, 2)])  # Start row is revealed
    }
    save_user_data()
    
    # Display initial grid
    display = get_grid_display(grid, current_row, current_col, user_data[user_id]['game_state']['revealed'])
    display += f"\n💰 Your Coins: {user_data[user_id]['coins']:,}\n"
    display += "\nChoose your next move:"
    
    keyboard = get_movement_keyboard(current_row, current_col)
    await update.message.reply_text(display, reply_markup=keyboard)

async def handle_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle button callbacks"""
    query = update.callback_query
    await query.answer()
    
    user_id = query.from_user.id
    init_user(user_id)
    
    if user_data[user_id].get('game_state') is None:
        await query.edit_message_text("No active game! Use /game to start.")
        return
    
    game_state = user_data[user_id]['game_state']
    grid = game_state['grid']
    current_row = game_state['current_row']
    current_col = game_state['current_col']
    revealed = game_state['revealed']
    # Ensure revealed is a set (in case it was loaded as a list)
    if isinstance(revealed, list):
        revealed = set(tuple(item) for item in revealed)
        game_state['revealed'] = revealed
    
    if query.data == "new_game":
        # Start new game
        grid = generate_grid()
        current_row = 0
        current_col = 1
        revealed = set([(0, 0), (0, 1), (0, 2)])
        
        game_state['grid'] = grid
        game_state['current_row'] = current_row
        game_state['current_col'] = current_col
        game_state['revealed'] = revealed
        save_user_data()
        
        display = get_grid_display(grid, current_row, current_col, revealed)
        display += f"\n💰 Your Coins: {user_data[user_id]['coins']:,}\n"
        display += "\nNew game started! Choose your next move:"
        
        keyboard = get_movement_keyboard(current_row, current_col)
        await query.edit_message_text(display, reply_markup=keyboard)
        return
    
    if query.data == "position":
        display = f"📍 Your Position:\nRow: {current_row + 1}/{GRID_ROWS}\nColumn: {current_col + 1}/{GRID_COLS}"
        await query.answer(display, show_alert=True)
        return
    
    if query.data == "coins":
        display = f"💰 Your Coins: {user_data[user_id]['coins']:,}"
        await query.answer(display, show_alert=True)
        return
    
    if query.data.startswith("move_"):
        # Parse movement
        parts = query.data.split("_")
        new_row = int(parts[1])
        new_col = int(parts[2])
        
        # Check if move is valid
        if new_row != current_row + 1 or new_col < 0 or new_col >= GRID_COLS:
            await query.answer("Invalid move!", show_alert=True)
            return
        
        # Check if hit a mine
        if grid[new_row][new_col] == 'mine':
            # Game over
            revealed.add((new_row, new_col))
            game_state['revealed'] = revealed
            save_user_data()
            
            display = get_grid_display(grid, new_row, new_col, revealed)
            display += "\n💣💣💣 BOOM! You hit a mine! 💣💣💣\n\n"
            display += "Game Over! Use /game to start a new game."
            
            keyboard = InlineKeyboardMarkup([
                [InlineKeyboardButton("🔄 New Game", callback_data="new_game")]
            ])
            await query.edit_message_text(display, reply_markup=keyboard)
            return
        
        # Valid move
        current_row = new_row
        current_col = new_col
        revealed.add((current_row, current_col))
        
        # Check if reached the end
        if current_row == GRID_ROWS - 1:
            # Victory!
            user_data[user_id]['coins'] += COIN_REWARD
            game_state['current_row'] = current_row
            game_state['current_col'] = current_col
            game_state['revealed'] = revealed
            save_user_data()
            
            display = get_grid_display(grid, current_row, current_col, revealed)
            display += "\n🎉🎉🎉 VICTORY! 🎉🎉🎉\n\n"
            display += f"You reached the end!\n"
            display += f"💰 Reward: +{COIN_REWARD:,} coins\n"
            display += f"💰 Total Coins: {user_data[user_id]['coins']:,}\n\n"
            display += "Use /game to play again!"
            
            keyboard = InlineKeyboardMarkup([
                [InlineKeyboardButton("🔄 New Game", callback_data="new_game")]
            ])
            await query.edit_message_text(display, reply_markup=keyboard)
            return
        
        # Continue game
        game_state['current_row'] = current_row
        game_state['current_col'] = current_col
        game_state['revealed'] = revealed
        save_user_data()
        
        display = get_grid_display(grid, current_row, current_col, revealed)
        display += f"\n💰 Your Coins: {user_data[user_id]['coins']:,}\n"
        display += "\nChoose your next move:"
        
        keyboard = get_movement_keyboard(current_row, current_col)
        await query.edit_message_text(display, reply_markup=keyboard)

async def coins(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Show user's coins"""
    user_id = update.effective_user.id
    init_user(user_id)
    
    msg = f"💰 Your Coins: {user_data[user_id]['coins']:,}"
    await update.message.reply_text(msg)

def main():
    """Start the bot"""
    # Load existing user data
    load_user_data()
    
    # Create application
    application = Application.builder().token(BOT_TOKEN).build()
    
    # Add handlers
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("game", game))
    application.add_handler(CommandHandler("coins", coins))
    application.add_handler(CallbackQueryHandler(handle_callback))
    
    # Start bot
    print("Bot is running...")
    application.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == "__main__":
    main()
