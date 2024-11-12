from flask import Flask, request, jsonify
from celery import Celery
import openai
import sendgrid
from sendgrid.helpers.mail import Mail, Email, To, Content
import pandas as pd
from google.oauth2.service_account import Credentials
from googleapiclient.discovery import build
import os
from dotenv import load_dotenv

# Load environment variables from .env file
load_dotenv()

# Initialize Flask app
app = Flask(__name__)

# Celery configuration
app.config['CELERY_BROKER_URL'] = os.getenv('CELERY_BROKER_URL', 'redis://localhost:6379/0')
celery = Celery(app.name, broker=app.config['CELERY_BROKER_URL'])
celery.conf.update(app.config)

# Google Sheets API authentication
SCOPES = ['https://www.googleapis.com/auth/spreadsheets.readonly']

# Initialize SendGrid API
SENDGRID_API_KEY = os.getenv('SENDGRID_API_KEY')

def send_email_via_sendgrid(to_email, subject, content):
    sg = sendgrid.SendGridAPIClient(api_key=SENDGRID_API_KEY)
    from_email = Email("youremail@example.com")
    to_email = To(to_email)
    content = Content("text/plain", content)
    mail = Mail(from_email, to_email, subject, content)
    response = sg.client.mail.send.post(request_body=mail.get())
    return response.status_code

# Google Sheets Integration
def authenticate_google_sheets():
    creds = Credentials.from_service_account_file('credentials.json', scopes=SCOPES)
    service = build('sheets', 'v4', credentials=creds)
    return service

def read_google_sheet(spreadsheet_id, range_name):
    service = authenticate_google_sheets()
    sheet = service.spreadsheets()
    result = sheet.values().get(spreadsheetId=spreadsheet_id, range=range_name).execute()
    values = result.get('values', [])
    return pd.DataFrame(values[1:], columns=values[0])  # Return as DataFrame

# Celery task to send emails
@celery.task
def send_custom_email(data):
    email_content = f"Dear {data['Company']},\nWe have great products like {data['Product']} for your location {data['Location']}."
    send_email_via_sendgrid(data['Email'], "Personalized Email", email_content)

@app.route('/send-emails', methods=['POST'])
def send_emails():
    file = request.files['file']
    df = pd.read_csv(file)
    for _, row in df.iterrows():
        send_custom_email.apply_async(args=[row.to_dict()])
    return jsonify({"status": "Emails are being sent in the background"}), 200

@app.route('/analytics', methods=['GET'])
def analytics():
    # Fetch real-time analytics (you can use Redis or a database here to store data)
    return jsonify({"emails_sent": 100, "emails_failed": 2}), 200

if __name__ == '__main__':
    app.run(debug=True)
from app import celery

if __name__ == '__main__':
    celery.start()
import React, { useState, useEffect } from 'react';
import axios from 'axios';

function App() {
  const [file, setFile] = useState(null);
  const [status, setStatus] = useState(null);
  const [emailStats, setEmailStats] = useState(null);

  const handleFileChange = (event) => {
    setFile(event.target.files[0]);
  };

  const handleUpload = async () => {
    const formData = new FormData();
    formData.append('file', file);

    try {
      const response = await axios.post('http://localhost:5000/send-emails', formData, {
        headers: { 'Content-Type': 'multipart/form-data' },
      });
      setStatus(response.data.status);
    } catch (error) {
      console.error(error);
      setStatus('Error occurred while sending emails.');
    }
  };

  useEffect(() => {
    const interval = setInterval(async () => {
      const result = await axios.get('http://localhost:5000/analytics');
      setEmailStats(result.data);
    }, 5000);

    return () => clearInterval(interval);
  }, []);

  return (
    <div className="App">
      <h1>Email Sender</h1>
      <input type="file" onChange={handleFileChange} />
      <button onClick={handleUpload}>Upload and Send Emails</button>
      {status && <p>Status: {status}</p>}
      {emailStats && (
        <div>
          <p>Emails Sent: {emailStats.emails_sent}</p>
          <p>Emails Failed: {emailStats.emails_failed}</p>
        </div>
      )}
    </div>
  );
}

export default App;
version: '3'

services:
  backend:
    build: ./backend
    ports:
      - "5000:5000"
    volumes:
      - ./backend:/app
    depends_on:
      - redis
    environment:
      - SENDGRID_API_KEY=${SENDGRID_API_KEY}
      - CELERY_BROKER_URL=redis://redis:6379/0
    command: python app.py

  redis:
    image: redis:latest
    ports:
      - "6379:6379"

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    volumes:
      - ./frontend:/app
    depends_on:
      - backend
{
  "name": "email-sender-app",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "axios": "^0.24.0",
    "react": "^17.0.2",
    "react-dom": "^17.0.2",
    "react-scripts": "^4.0.3"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  }
}





