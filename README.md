import telebot
import requests
import yt_dlp
import os
from flask import Flask

TOKEN = "7513143226:AAFabCnQMKGegYfv3IbEgq9weWlg_vNIAeE"
bot = telebot.TeleBot(TOKEN)

# Flask app to keep the bot alive on free hosting
app = Flask(__name__)

@app.route('/')
def home():
    return "Bot is Running!"

def download_video(url, platform):
    filename = f"{platform}_video.mp4"
    
    if platform == "instagram":
        os.system(f"instaloader --dirname-pattern=temp --filename-pattern={filename} {url}")
        return f"temp/{filename}"
    
    elif platform == "tiktok":
        api_url = f"https://api.snaptik.app/tiktok?url={url}"
        response = requests.get(api_url).json()
        if "video_url" in response:
            video_url = response["video_url"]
            with open(filename, "wb") as f:
                f.write(requests.get(video_url).content)
            return filename
    
    elif platform == "facebook":
        ydl_opts = {'outtmpl': filename}
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            ydl.download([url])
        return filename
    
    elif platform == "youtube":
        ydl_opts = {'format': 'best', 'outtmpl': filename}
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            ydl.download([url])
        return filename
    
    return None

@bot.message_handler(commands=['start'])
def send_welcome(message):
    bot.reply_to(message, "🎥 Welcome! Send a link from Instagram, Facebook, TikTok, or YouTube to download.")

@bot.message_handler(func=lambda message: True)
def handle_message(message):
    url = message.text
    platform = None

    if "instagram.com" in url:
        platform = "instagram"
    elif "tiktok.com" in url:
        platform = "tiktok"
    elif "facebook.com" in url:
        platform = "facebook"
    elif "youtube.com" in url or "youtu.be" in url:
        platform = "youtube"

    if platform:
        bot.reply_to(message, "⏳ Downloading your video, please wait...")
        video_file = download_video(url, platform)

        if video_file:
            with open(video_file, "rb") as video:
                bot.send_video(message.chat.id, video)
            os.remove(video_file)
        else:
            bot.reply_to(message, "⚠️ Failed to download the video.")
    else:
        bot.reply_to(message, "⚠️ Invalid link. Please send a link from YouTube, Facebook, Instagram, or TikTok.")

print("Bot is running...")
bot.polling()

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
