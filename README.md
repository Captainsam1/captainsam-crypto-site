// index.js - Run with `node index.js`

import express from 'express';
import cors from 'cors';
import bodyParser from 'body-parser';
import path from 'path';
import { fileURLToPath } from 'url';
import axios from 'axios';
import { ethers } from 'ethers';
import dotenv from 'dotenv';

dotenv.config();

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

const app = express();
app.use(cors());
app.use(bodyParser.json());

// Ethers wallet & provider setup
const provider = new ethers.providers.JsonRpcProvider(process.env.ALCHEMY_API || '<ALCHEMY_API_URL>');
const wallet = new ethers.Wallet(process.env.PRIVATE_KEY || '<PRIVATE_KEY>', provider);

// Serve React frontend static files (make sure to build your frontend in /client/build before)
app.use(express.static(path.join(__dirname, 'client/build')));

app.get('/', (req, res) => res.sendFile(path.join(__dirname, 'client/build', 'index.html')));

// MPESA STK Push simulation endpoint
app.post('/api/mpesa/stkpush', async (req, res) => {
  const { phone, amount } = req.body;
  console.log(`STK push requested for phone: ${phone}, amount: ${amount} KES`);

  try {
    // Simulated MPESA STK Push response (replace with real API logic for production)
    res.json({
      MerchantRequestID: '123456',
      CheckoutRequestID: 'abc123',
      ResponseCode: '0',
      ResponseDescription: `STK Push simulated for ${amount} KES to ${phone}`,
      CustomerMessage: 'Please check your phone to confirm payment.',
    });
  } catch (error) {
    res.status(500).json({ error: 'Error in MPESA STK Push' });
  }
});

// Blockchain transaction relay simulation endpoint
app.post('/api/transaction/paygas', async (req, res) => {
  try {
    const tx = {
      to: req.body.to,
      data: req.body.data,
      gasLimit: req.body.gasLimit || 21000,
      gasPrice: req.body.gasPrice ? ethers.utils.parseUnits(req.body.gasPrice, 'gwei') : ethers.utils.parseUnits('10', 'gwei'),
    };
    const sent = await wallet.sendTransaction(tx);
    await sent.wait();
    res.json({ txHash: sent.hash });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Serve inline React app for demo (remove if you use a separate frontend)
app.get('/client/build/index.html', (req, res) => {
  res.send(`
  <!DOCTYPE html>
  <html lang="en">
  <head><meta charset="UTF-8" /><title>Gas Fee Purchase</title></head>
  <body>
  <div id="root"></div>
  <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
  <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
  <script crossorigin src="https://unpkg.com/axios/dist/axios.min.js"></script>
  <script>
    const e = React.createElement;

    function App() {
      const [phone, setPhone] = React.useState('');
      const [amount, setAmount] = React.useState('');
      const [msg, setMsg] = React.useState('');

      async function handleSubmit(e) {
        e.preventDefault();
        try {
          const res = await axios.post('/api/mpesa/stkpush', { phone, amount });
          setMsg(res.data.CustomerMessage);
        } catch(err) {
          setMsg('Error making payment');
        }
      }

      return e('div', null,
        e('h1', null, 'Buy Gas Fee with MPESA'),
        e('form', { onSubmit: handleSubmit },
          e('input', { type: 'text', placeholder: 'Phone Number', value: phone, onChange: e => setPhone(e.target.value) }),
          e('input', { type: 'number', placeholder: 'Amount (KES)', value: amount, onChange: e => setAmount(e.target.value) }),
          e('button', { type: 'submit' }, 'Pay')
        ),
        e('p', null, msg)
      );
    }

    ReactDOM.createRoot(document.getElementById('root')).render(e(App));
  </script>
  </body>
  </html>
  `);
});

// Start server
const PORT = process.env.PORT || 4000;
app.listen(PORT, () => console.log(`Running on port ${PORT}`));
