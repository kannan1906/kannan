import tkinter as tk
from tkinter import ttk
import cv2
from PIL import Image, ImageTk
from deepface import DeepFace
import threading
import time
import requests
import base64
import json
import vlc  # Using VLC for audio playback
from typing import Dict, List
import logging

class SpotifyClient:
    def __init__(self):
        self.CLIENT_ID = "b30493cbdc054c459d7d5177d1ef4fa6"
        self.CLIENT_SECRET = "52252a2f798e4946840b585c1306b266"
        self.access_token = None
        self.token_expiry = 0
        
    def get_access_token(self) -> str:
        if self.access_token and time.time() < self.token_expiry:
            return self.access_token
            
        auth_string = base64.b64encode(f"{self.CLIENT_ID}:{self.CLIENT_SECRET}".encode()).decode()
        auth_url = "https://accounts.spotify.com/api/token"
        headers = {"Authorization": f"Basic {auth_string}"}
        data = {"grant_type": "client_credentials"}
        
        response = requests.post(auth_url, headers=headers, data=data)
        if response.status_code == 200:
            token_data = response.json()
            self.access_token = token_data["access_token"]
            self.token_expiry = time.time() + token_data["expires_in"]
            return self.access_token
        return None

    def get_tracks_by_emotion(self, emotion: str) -> List[Dict]:
        token = self.get_access_token()
        if not token:
            return []

        emotion_genres = {
            'happy': 'happy',
            'sad': 'sad',
            'angry': 'aggressive',
            'neutral': 'chill',
            'surprise': 'party',
            'fear': 'ambient',
            'disgust': 'rock'
        }
        
        genre = emotion_genres.get(emotion.lower(), 'pop')
        search_url = "https://api.spotify.com/v1/search"
        headers = {"Authorization": f"Bearer {token}"}
        params = {
            "q": f"genre:{genre}",
            "type": "track",
            "limit": 10
        }
        
        response = requests.get(search_url, headers=headers, params=params)
        if response.status_code == 200:
            tracks = response.json()["tracks"]["items"]
            return [{
                'name': track['name'],
                'artist': track['artists'][0]['name'],
                'preview_url': track['preview_url']
            } for track in tracks if track['preview_url']]
        return []

class EmotionMusicPlayer:
    def __init__(self):
        self.root = tk.Tk()
        self.root.title("Real-time Emotion Music Player")
        self.root.geometry("1000x800")
        
        # Initialize components
        self.setup_gui()
        self.setup_camera()
        
        # Initialize Spotify client and VLC player
        self.spotify = SpotifyClient()
        self.player = vlc.MediaPlayer()
        
        # State variables
        self.current_emotion = None
        self.current_tracks = []
        self.current_track_index = 0
        self.is_playing = False
        self.last_emotion_check = 0
        self.emotion_check_interval = 3  # seconds
        
    def setup_gui(self):
        # Main container
        self.main_frame = ttk.Frame(self.root, padding="10")
        self.main_frame.grid(row=0, column=0, sticky=(tk.W, tk.E, tk.N, tk.S))
        
        # Camera frame
        self.camera_label = ttk.Label(self.main_frame)
        self.camera_label.grid(row=0, column=0, columnspan=2, pady=10)
        
        # Emotion display
        self.emotion_label = ttk.Label(self.main_frame, text="Detected Emotion: None", font=('Arial', 12))
        self.emotion_label.grid(row=1, column=0, columnspan=2, pady=10)
        
        # Music controls
        self.controls_frame = ttk.Frame(self.main_frame)
        self.controls_frame.grid(row=2, column=0, columnspan=2, pady=10)
        
        self.prev_button = ttk.Button(self.controls_frame, text="Previous", command=self.play_previous)
        self.prev_button.grid(row=0, column=0, padx=5)
        
        self.play_button = ttk.Button(self.controls_frame, text="Play", command=self.toggle_play)
        self.play_button.grid(row=0, column=1, padx=5)
        
        self.next_button = ttk.Button(self.controls_frame, text="Next", command=self.play_next)
        self.next_button.grid(row=0, column=2, padx=5)
        
        # Playlist
        self.playlist_frame = ttk.Frame(self.main_frame)
        self.playlist_frame.grid(row=3, column=0, columnspan=2, pady=10, sticky=(tk.W, tk.E))
        
        self.playlist_label = ttk.Label(self.playlist_frame, text="Current Playlist", font=('Arial', 12))
        self.playlist_label.pack()
        
        self.playlist_listbox = tk.Listbox(self.playlist_frame, height=10, width=50)
        self.playlist_listbox.pack(fill=tk.BOTH, expand=True)
        self.playlist_listbox.bind('<<ListboxSelect>>', self.on_track_selected)
        
    def setup_camera(self):
        self.cap = cv2.VideoCapture(0)
        self.face_cascade = cv2.CascadeClassifier(
            cv2.data.haarcascades + 'haarcascade_frontalface_default.xml'
        )
        self.update_camera()
        
    def update_camera(self):
        ret, frame = self.cap.read()
        if ret:
            # Detect face and emotion in a separate thread
            threading.Thread(target=self.process_frame, args=(frame,)).start()
            
            # Convert frame for display
            frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
            img = Image.fromarray(frame)
            img = img.resize((640, 480))
            imgtk = ImageTk.PhotoImage(image=img)
            self.camera_label.imgtk = imgtk
            self.camera_label.configure(image=imgtk)
        
        self.root.after(10, self.update_camera)
        
    def process_frame(self, frame):
        current_time = time.time()
        if current_time - self.last_emotion_check < self.emotion_check_interval:
            return
            
        try:
            result = DeepFace.analyze(frame, actions=['emotion'], enforce_detection=False)
            if isinstance(result, list):
                result = result[0]
            
            emotion = result['dominant_emotion']
            if emotion != self.current_emotion:
                self.current_emotion = emotion
                self.emotion_label.config(text=f"Detected Emotion: {emotion}")
                self.update_playlist(emotion)
            
            self.last_emotion_check = current_time
            
        except Exception as e:
            logging.error(f"Error detecting emotion: {str(e)}")
            
    def update_playlist(self, emotion):
        # Get tracks for the emotion
        tracks = self.spotify.get_tracks_by_emotion(emotion)
        if tracks:
            self.current_tracks = tracks
            self.current_track_index = 0
            
            # Update playlist display
            self.playlist_listbox.delete(0, tk.END)
            for track in tracks:
                self.playlist_listbox.insert(tk.END, f"{track['name']} - {track['artist']}")
            
            # Start playing first track
            self.play_current_track()
            
    def play_current_track(self):
        if not self.current_tracks:
            return
            
        track = self.current_tracks[self.current_track_index]
        if track['preview_url']:
            self.player.set_mrl(track['preview_url'])
            self.player.play()
            self.is_playing = True
            self.play_button.config(text="Pause")
        else:
            self.emotion_label.config(text="No audio preview available")
            
    def toggle_play(self):
        if self.is_playing:
            self.player.pause()
            self.play_button.config(text="Play")
        else:
            self.player.play()
            self.play_button.config(text="Pause")
        self.is_playing = not self.is_playing
        
    def play_next(self):
        if not self.current_tracks:
            return
            
        self.current_track_index = (self.current_track_index + 1) % len(self.current_tracks)
        self.play_current_track()
        
    def play_previous(self):
        if not self.current_tracks:
            return
            
        self.current_track_index = (self.current_track_index - 1) % len(self.current_tracks)
        self.play_current_track()
        
    def on_track_selected(self, event):
        selection = self.playlist_listbox.curselection()
        if selection:
            self.current_track_index = selection[0]
            self.play_current_track()
            
    def run(self):
        self.root.mainloop()
        
    def cleanup(self):
        self.cap.release()
        self.player.stop()

if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    player = EmotionMusicPlayer()
    try:
        player.run()
    finally:
        player.cleanup()
