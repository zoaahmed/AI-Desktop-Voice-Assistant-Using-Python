# AI-Desktop-Voice-Assistant-Using-Python
"""
Jarvis AI Desktop Voice Assistant — OpenAI Chat Version
=========================================================
This version adds persistent conversation memory using OpenAI GPT
and saves AI-generated responses to the Openai/ directory.

Author: Zoa Ahmed
Institution: B.P. Poddar Institute of Management & Technology
"""

import speech_recognition as sr
import os
import webbrowser
import openai
import datetime
import random
import requests
import smtplib

try:
    from config import apikey, gmail_address, gmail_app_password, recipient_email
except ImportError:
    apikey             = "YOUR_OPENAI_API_KEY"
    gmail_address      = "youremail@gmail.com"
    gmail_app_password = "your-app-password"
    recipient_email    = "recipient@example.com"

# Persistent conversation string for GPT context
chatStr = ""


# ── Core Utilities ─────────────────────────────────────────────────────────────

def say(text: str) -> None:
    """Speak text aloud (macOS). For Windows/Linux replace with pyttsx3."""
    os.system(f'say "{text}"')


def takeCommand() -> str:
    """Listen via microphone and return recognised text."""
    r = sr.Recognizer()
    with sr.Microphone() as source:
        try:
            print("Listening...")
            audio = r.listen(source, timeout=5, phrase_time_limit=10)
            print("Recognizing...")
            query = r.recognize_google(audio, language="en-in")
            print(f"User said: {query}")
            return query
        except Exception as e:
            print(e)
            return "Some Error Occurred. Sorry from Jarvis"


# ── OpenAI Integration ────────────────────────────────────────────────────────

def chat(query: str) -> str:
    """
    Send a query to OpenAI GPT and maintain conversation context.
    Speaks and returns the response.
    """
    global chatStr
    openai.api_key = apikey
    chatStr += f"Harry: {query}\n Jarvis: "
    response = openai.Completion.create(
        model="text-davinci-003",
        prompt=chatStr,
        temperature=0.7,
        max_tokens=256,
        top_p=1,
        frequency_penalty=0,
        presence_penalty=0,
    )
    reply = response["choices"][0]["text"]
    say(reply)
    chatStr += f"{reply}\n"
    return reply


def ai(prompt: str) -> None:
    """
    Send a standalone prompt to OpenAI GPT and save the result to
    the Openai/ directory as a text file named after the prompt.
    """
    openai.api_key = apikey
    text = f"OpenAI response for Prompt: {prompt} \n *************************\n\n"
    response = openai.Completion.create(
        model="text-davinci-003",
        prompt=prompt,
        temperature=0.7,
        max_tokens=256,
        top_p=1,
        frequency_penalty=0,
        presence_penalty=0,
    )
    text += response["choices"][0]["text"]

    os.makedirs("Openai", exist_ok=True)
    filename = "".join(prompt.split("intelligence")[1:]).strip() or "response"
    with open(f"Openai/{filename}.txt", "w") as f:
        f.write(text)


# ── Optional API Features ─────────────────────────────────────────────────────

def get_weather(city: str) -> str:
    """Fetch current weather for a city using OpenWeatherMap."""
    try:
        from config import weather_api_key
    except ImportError:
        weather_api_key = "YOUR_OPENWEATHERMAP_API_KEY"

    base_url = "http://api.openweathermap.org/data/2.5/weather"
    params   = {"q": city, "appid": weather_api_key, "units": "metric"}
    try:
        data        = requests.get(base_url, params=params).json()
        description = data["weather"][0]["description"]
        temp        = data["main"]["temp"]
        return f"The weather in {city} is {description} with a temperature of {temp}°C."
    except Exception:
        return "Sorry, I couldn't fetch the weather information."


def get_news() -> str:
    """Fetch top headlines using News API."""
    try:
        from config import news_api_key
    except ImportError:
        news_api_key = "YOUR_NEWS_API_KEY"

    base_url = "https://newsapi.org/v2/top-headlines"
    params   = {"country": "in", "apiKey": news_api_key}
    try:
        data      = requests.get(base_url, params=params).json()
        articles  = data["articles"]
        headlines = "\n".join(
            [f"{i + 1}. {a['title']}" for i, a in enumerate(articles[:5])]
        )
        return f"Here are the top headlines:\n{headlines}"
    except Exception:
        return "Sorry, I couldn't fetch the news."


def sendEmail(to: str, content: str) -> None:
    """Send an email via Gmail SMTP."""
    server = smtplib.SMTP("smtp.gmail.com", 587)
    server.ehlo()
    server.starttls()
    server.login(gmail_address, gmail_app_password)
    server.sendmail(gmail_address, to, content)
    server.close()


# ── Main Loop ──────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    print("Welcome to Jarvis A.I")
    say("Jarvis A.I")

    SITES = [
        ["youtube",   "https://www.youtube.com"],
        ["wikipedia", "https://www.wikipedia.com"],
        ["google",    "https://www.google.com"],
    ]

    while True:
        print("Listening...")
        query = takeCommand()

        for site in SITES:
            if f"open {site[0]}".lower() in query.lower():
                say(f"Opening {site[0]} sir...")
                webbrowser.open(site[1])

        if "open music" in query:
            music_path = "/Users/harry/Downloads/downfall-21371.mp3"
            os.system(f"open {music_path}")

        elif "the time" in query:
            hour = datetime.datetime.now().strftime("%H")
            mins = datetime.datetime.now().strftime("%M")
            say(f"Sir time is {hour} bajke {mins} minutes")

        elif "open facetime" in query.lower():
            os.system("open /System/Applications/FaceTime.app")

        elif "using artificial intelligence" in query.lower():
            ai(prompt=query)

        elif "check weather" in query.lower():
            say("Sure, please specify the city.")
            city = takeCommand().lower()
            say(get_weather(city))

        elif "check news" in query.lower():
            say(get_news())

        elif "open code" in query:
            code_path = r"C:\Users\Haris\AppData\Local\Programs\Microsoft VS Code\Code.exe"
            os.startfile(code_path)

        elif "email to harry" in query:
            try:
                say("What should I say?")
                content = takeCommand()
                sendEmail("harryyouremail@gmail.com", content)
                say("Email has been sent!")
            except Exception as e:
                print(e)
                say("Sorry, I am not able to send this email.")

        elif "jarvis quit" in query.lower():
            say("Goodbye!")
            exit()

        elif "reset chat" in query.lower():
            chatStr = ""
            say("Chat history cleared.")

        else:
            print("Chatting...")
            chat(query)
