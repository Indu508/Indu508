js
const express = require('express');
const cors = require('cors');
const bodyParser = require('body-parser');
const emailRoutes = require('./routes/emailRoutes');
const analyticsRoutes = require('./routes/analyticsRoutes');

const app = express();

app.use(cors());
app.use(bodyParser.json());

// Routes
app.use('/api/emails', emailRoutes);
app.use('/api/analytics', analyticsRoutes);

// Start the server
const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
emailRoutes.js - The routes for email sending and scheduling.
js
const express = require('express');
const { sendEmails, scheduleEmails, getEmailStatus } = require('../controllers/emailController');

const router = express.Router();

router.post('/send', sendEmails); // Endpoint to send emails
router.post('/schedule', scheduleEmails); // Endpoint to schedule emails
router.get('/status/:emailId', getEmailStatus); // Get the status of a specific email

module.exports = router;
emailController.js - Handles the logic for sending emails, scheduling them, and tracking status.
js
const { sendEmail } = require('../services/emailService');
const { scheduleEmail } = require('../services/throttlingService');
const Email = require('../models/emailModel');

// Send email using SendGrid
exports.sendEmails = async (req, res) => {
    const { data, template, emailConfig } = req.body;
    
    try {
        // Loop through the dataset and send personalized emails
        for (const row of data) {
            let personalizedContent = template;
            for (const column in row) {
                personalizedContent = personalizedContent.replace(`{${column}}`, row[column]);
            }

            // Call SendGrid API or SMTP to send the email
            await sendEmail(row.Email, personalizedContent, emailConfig);
        }
        res.status(200).json({ message: 'Emails sent successfully' });
    } catch (error) {
        console.error(error);
        res.status(500).json({ message: 'Error sending emails', error });
    }
};

// Schedule emails with throttling
exports.scheduleEmails = async (req, res) => {
    const { data, template, emailConfig, scheduleTime } = req.body;
    
    try {
        // Schedule the emails at the provided time, with throttling
        scheduleEmail(data, template, emailConfig, scheduleTime);
        res.status(200).json({ message: 'Emails scheduled successfully' });
    } catch (error) {
        console.error(error);
        res.status(500).json({ message: 'Error scheduling emails', error });
    }
};

// Get the status of a specific email
exports.getEmailStatus = async (req, res) => {
    const { emailId } = req.params;
    
    try {
        const emailStatus = await Email.findById(emailId);
        res.status(200).json(emailStatus);
    } catch (error) {
        console.error(error);
        res.status(500).json({ message: 'Error retrieving email status', error });
    }
};
js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { Bar } from 'react-chartjs-2';
import { toast } from 'react-toastify';

const Dashboard = () => {
  const [emailStatus, setEmailStatus] = useState([]);
  const [analytics, setAnalytics] = useState({ totalSent: 0, totalFailed: 0 });

  useEffect(() => {
    const fetchEmailStatus = async () => {
      try {
        const res = await axios.get('/api/analytics/status');
        setEmailStatus(res.data);
      } catch (error) {
        console.error(error);
        toast.error('Failed to fetch email statuses');
      }
    };

    const fetchAnalytics = async () => {
      try {
        const res = await axios.get('/api/analytics/summary');
        setAnalytics(res.data);
      } catch (error) {
        console.error(error);
        toast.error('Failed to fetch analytics');
      }
    };

    fetchEmailStatus();
    fetchAnalytics();
  }, []);

  return (
    <div>
      <h1>Email Dashboard</h1>
      <Bar
        data={{
          labels: ['Sent', 'Failed'],
          datasets: [{
            label: 'Email Stats',
            data: [analytics.totalSent, analytics.totalFailed],
            backgroundColor: ['#4caf50', '#f44336'],
          }]
        }}
      />
      <table>
        <thead>
          <tr>
            <th>Email</th>
            <th>Status</th>
            <th>Delivery Status</th>
          </tr>
        </thead>
        <tbody>
          {emailStatus.map((email) => (
            <tr key={email.id}>
              <td>{email.email}</td>
              <td>{email.status}</td>
              <td>{email.deliveryStatus}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
};

export default Dashboard;
js
const sgMail = require('@sendgrid/mail');
sgMail.setApiKey(process.env.SENDGRID_API_KEY);

// Send email using SendGrid
const sendEmail = async (to, content, emailConfig) => {
    const msg = {
        to,
        from: emailConfig.fromEmail,
        subject: emailConfig.subject,
        html: content,
    };

    try {
        await sgMail.send(msg);
    } catch (error) {
        console.error(error);


