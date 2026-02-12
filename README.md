# Bước 1: Lấy Token từ Telegram
1. Mở ứng dụng Telegram, tìm kiếm từ khóa: **@BotFather**
2. Bấm **Start**
3. Gõ lệnh: `/newbot`
4. **Đặt tên hiển thị:** gì cx đc 
5. **Đặt username:** (Phải kết thúc bằng chữ `bot` và không trùng với ai).
6. Nếu thành công, BotFather sẽ đưa cho bạn một đoạn mã dài dạng: `7123456:AAH...`. 

# Bước 2: Cài đặt công cụ cần thiết

Bạn cần cài Python và các thư viện hỗ trợ.

1. **Cài Python:** Tải tại [python.org](https://www.python.org/) và cài đặt Nhớ *Add Python to PATH*
2. **Cài thư viện:** Mở **CMD** (Command Prompt) hoặc **Terminal** điền

```bash
pip install python-telegram-bot aiohttp

```

### Bước 3: Viết Code 

Bạn tạo một file mới tên là `bot_phim.py` (dùng Notepad, VS Code hoặc PyCharm đều được), sau đó dán toàn bộ đoạn code dưới đây vào.

**Lưu ý:** Thay dòng `TOKEN_CUA_BAN_O_DAY` bằng cái mã bạn vừa lấy ở Bước 1.

```python
import logging
import aiohttp
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.constants import ParseMode
from telegram.ext import ApplicationBuilder, ContextTypes, CommandHandler, MessageHandler, CallbackQueryHandler, filters

# ================= CẤU HÌNH =================
# 1. Dán Token của bạn vào giữa hai dấu nháy đơn
BOT_TOKEN = 'TOKEN_CUA_BAN_O_DAY' 

# 2. API của web phim (Ophim / Nguonc)
API_DOMAIN = "https://ophim1.com"

# Thiết lập log để báo lỗi nếu có
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)

# ================= HÀM GỌI API (LẤY DỮ LIỆU) =================

async def search_phim(keyword):
    """Tìm phim theo tên, trả về danh sách tên + slug"""
    # URL tìm kiếm chuẩn của Ophim
    url = f"{API_DOMAIN}/v1/api/tim-kiem?keyword={keyword}"
    
    async with aiohttp.ClientSession() as session:
        try:
            async with session.get(url) as resp:
                if resp.status == 200:
                    data = await resp.json()
                    # Trả về danh sách phim từ JSON
                    return data.get('data', {}).get('items', [])
        except Exception as e:
            print(f"Lỗi tìm kiếm: {e}")
    return []

async def get_link_m3u8(slug):
    """Lấy chi tiết tập phim từ slug"""
    # URL lấy chi tiết phim
    url = f"{API_DOMAIN}/v1/api/kho-phim/{slug}"
    
    async with aiohttp.ClientSession() as session:
        try:
            async with session.get(url) as resp:
                if resp.status == 200:
                    return await resp.json()
        except Exception as e:
            print(f"Lỗi lấy link: {e}")
    return None

# ================= XỬ LÝ TIN NHẮN TỪ NGƯỜI DÙNG =================

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("👋 Chào! Hãy gõ tên phim bạn muốn xem (Ví dụ: Mai, Dao, Pho).")

async def handle_tim_kiem(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_text = update.message.text
    
    # Báo đang tìm...
    msg = await update.message.reply_text(f"🔍 Đang tìm: {user_text}...")
    
    # Gọi hàm tìm kiếm
    movies = await search_phim(user_text)
    
    if not movies:
        await msg.edit_text("❌ Không tìm thấy phim nào. Thử tên khác xem sao?")
        return

    # Tạo danh sách nút bấm
    keyboard = []
    for movie in movies:
        name = movie['name']
        slug = movie['slug']
        year = movie.get('year', '')
        
        # Nút bấm sẽ chứa data dạng: "xem|slug-cua-phim"
        # (Lưu ý: Telegram giới hạn data nút bấm 64 ký tự, slug dài quá có thể lỗi)
        keyboard.append([InlineKeyboardButton(f"🎬 {name} ({year})", callback_data=f"xem|{slug}")])
    
    # Giới hạn chỉ hiện 5 phim đầu tiên để đẹp giao diện
    if len(keyboard) > 5:
        keyboard = keyboard[:5]
    
    reply_markup = InlineKeyboardMarkup(keyboard)
    await msg.edit_text(f"✅ Tìm thấy {len(movies)} phim. Chọn phim để lấy link:", reply_markup=reply_markup)

async def handle_nut_bam(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer() # Xác nhận đã bấm
    
    data = query.data # Lấy dữ liệu từ nút (vd: xem|dao-pho-va-piano)
    
    if data.startswith("xem|"):
        slug = data.split("|")[1]
        
        await query.edit_message_text(f"⏳ Đang lấy link phim: {slug}...")
        
        # Gọi API lấy tập phim
        movie_data = await get_link_m3u8(slug)
        
        if not movie_data or 'episodes' not in movie_data:
            await query.edit_message_text("❌ Lỗi: Không lấy được danh sách tập.")
            return
            
        # Xử lý kết quả
        movie_info = movie_data['movie']
        episodes = movie_data['episodes']
        
        text_hien_thi = f"🎥 **{movie_info['name']}**\n"
        text_hien_thi += f"📅 Năm: {movie_info.get('year', 'N/A')} | ⭐ {movie_info.get('lang', 'N/A')}\n"
        text_hien_thi += "👇 **Bấm vào link dưới để xem:**\n\n"
        
        co_tap_phim = False
        
        # Duyệt qua các server (Thường server đầu tiên là ngon nhất)
        for server in episodes:
            server_name = server['server_name']
            server_data = server['server_data']
            
            # Tạo list link: [Tập 1](link) [Tập 2](link)
            links = []
            for ep in server_data:
                name = ep['name'] # Tên tập (1, 2, Full)
                link = ep['link_m3u8'] # Link m3u8
                links.append(f"[{name}]({link})")
                co_tap_phim = True
            
            # Nối các link lại với nhau
            text_hien_thi += f"📡 Server {server_name}: " + " | ".join(links) + "\n\n"
            
        if not co_tap_phim:
            text_hien_thi += "❌ Phim này đang cập nhật, chưa có link."

        # Gửi kết quả (Dùng disable_web_page_preview=True để tin nhắn gọn hơn)
        await context.bot.send_message(
            chat_id=query.message.chat_id,
            text=text_hien_thi,
            parse_mode=ParseMode.MARKDOWN,
            disable_web_page_preview=True
        )

# ================= CHẠY BOT =================
if __name__ == '__main__':
    print("🤖 Bot đang khởi động...")
    app = ApplicationBuilder().token(BOT_TOKEN).build()
    
    # Lệnh /start
    app.add_handler(CommandHandler("start", start))
    
    # Xử lý tin nhắn văn bản (Tìm kiếm)
    app.add_handler(MessageHandler(filters.TEXT & (~filters.COMMAND), handle_tim_kiem))
    
    # Xử lý nút bấm
    app.add_handler(CallbackQueryHandler(handle_nut_bam))
    
    print("✅ Bot đã sẵn sàng! Vào Telegram test thôi.")
    app.run_polling()

```

# Bước 4: Chạy Bot và Hưởng thụ

1. Mở CMD tại thư mục chứa file `bot_phim.py`.
2. Gõ lệnh: `python bot_phim.py`.
3. Nếu thấy dòng chữ **"✅ Bot đã sẵn sàng!..."** là thành công.
4. Vào Telegram, chat với con bot của bạn:
* Gõ tên phim: `Lật mặt`
* Nó sẽ hiện danh sách nút.
* Bấm vào nút -> Nó nhả ra link M3U8.



### Mẹo nhỏ khi dùng:
Nếu muốn chạy cả ngày phải thuê VPS 
