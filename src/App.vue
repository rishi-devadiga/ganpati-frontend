<template>
  <div class="container py-5">
    <div class="row justify-content-end">
      <div class="col-md-8 col-lg-6 ms-auto">
        <div class="card shadow-lg border-0">
          <div class="card-header bg-warning text-center">
            <img src="/receiptLogo.png" alt="Samiti Logo" class="mb-2" style="height:100px;">
            <h2 class="mb-0">Sankalp Utsav Samiti</h2>
            <p class="mb-0">Poonam Sagar Cha Raja</p>
          </div>
          <div v-if="loading" class="loading-overlay">
            <div class="spinner-border text-primary" role="status">
              <span class="visually-hidden">Loading...</span>
            </div>
          </div>
          <div class="card-body bg-light">
            <form>
              <div class="mb-3">
                <label class="form-label">Name</label>
                <input v-model="form.name" class="form-control" placeholder="Your Name" required />
              </div>
              <div class="mb-3">
                <label class="form-label">Address</label>
                <input v-model="form.address" class="form-control" placeholder="Your Address" required />
              </div>
              <div class="mb-3">
                <label class="form-label">Phone</label>
                <input v-model="form.phone" class="form-control" placeholder="Your Phone" required />
              </div>
              <div class="mb-3">
                <label class="form-label">Email</label>
                <input v-model="form.email" type="email" class="form-control" placeholder="Your Email" required />
              </div>
              <div class="mb-3">
                <label class="form-label">Transaction Type</label>
                <select v-model="form.transaction_type" class="form-select" required>
                  <option value="" disabled>Select Type</option>
                  <option value="cash">Cash</option>
                  <option value="online">Online</option>
                </select>
              </div>
              <div class="mb-3">
                <label class="form-label">Amount (INR)</label>
                <input v-model="form.amount" type="number" min="1" class="form-control" placeholder="Amount" required />
              </div>
              <!-- Status dropdown only if cash -->
              <div v-if="form.transaction_type === 'cash'">
                <label>Status</label>
                <select v-model="status" class="form-select" required>
                  <option value="pending">Pending</option>
                  <option value="completed">Completed</option>
                </select>
              </div>
              <div class="d-flex justify-content-between">
                <button v-if="form.transaction_type === 'online'" type="button" class="btn btn-success" @click="payNow">
                  <i class="bi bi-credit-card"></i> Pay Now
                </button>
                <button v-if="form.transaction_type === 'cash'" type="button" class="btn btn-primary" @click="cashPay">
                  <i class="bi bi-cash"></i> Submit Cash Payment
                </button>
              </div>
            </form>
          </div>
          <div class="card-footer text-center bg-warning bg-opacity-25">
            <small class="text-muted">Thank you for your generous support. May Lord Ganesha bless you!</small>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
/* global Razorpay */
import { ref } from 'vue'
import jsPDF from 'jspdf'
import axios from 'axios'

const form = ref({
  name: '',
  address: '',
  phone: '',
  email: '',
  transaction_type: '',
  amount: 0
})
const data = ref({ status: '' }) // 1. Track payment status
const loading = ref(false) // 2. Track loading state
const status = ref('null');
const transaction_type = ref('null') // 5. Track status for cash payments

async function generateReceipt(data) {
  const doc = new jsPDF({orientation: 'landscape'});
  loading.value = true; // 3. Set loading state to true

  const logoUrl = '/receiptLogo.png';
  const logoData = await loadImageAsBase64(logoUrl);
  doc.addImage(logoData, 'PNG', 20, 20, 50, 50); // Adjust size and position as needed
  doc.setFontSize(20);
  doc.setTextColor(255, 0, 0);
  doc.setFont('Ancizar Serif', 'bold');
  doc.text('Sankalp Utsav Samiti', 100, 25);
  doc.text('Poonam Sagar Cha Raja', 100, 35);
  doc.text('________________________________________', 20, 45);

  doc.setFontSize(16);
  doc.setTextColor(0, 0, 0);
  doc.text(`Name: ${data.name}`, 20, 50);
  doc.text(`Address: ${data.address}`, 20, 60);
  doc.text(`Phone: ${data.phone}`, 20, 70);
  doc.text(`Email: ${data.email}`, 20, 80);
  doc.text(`Transaction Type: ${data.transaction_type}`, 20, 90);
  doc.text(`Amount: ₹ ${(data.amount / 100).toFixed(2)}`, 20, 100);
  doc.text(`Order ID: ${data.razorpay_order_id}`, 20, 110);
  doc.text(`Payment ID: ${data.razorpay_payment_id}`, 20, 120);
  doc.text(`Date: ${new Date().toLocaleString()}`, 20, 130);


  const pdfBlob = doc.output("blob");

  const formData = new FormData();
  formData.append("email", form.value.email);
  formData.append("pdf", pdfBlob, "receipt.pdf");

  try {
    await axios.post("https://ganpati-backend.onrender.com/send-receipt", formData, {
      headers: { "Content-Type": "multipart/form-data" },
    });
  } catch (error) {
    console.error("Failed to send:", error);
  }
  finally {
    loading.value = false; // 4. Reset loading state
  }
}
async function loadImageAsBase64(url) {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.crossOrigin = 'Anonymous';
    img.onload = () => {
      const canvas = document.createElement('canvas');
      canvas.width = img.width;
      canvas.height = img.height;
      const ctx = canvas.getContext('2d');
      ctx.drawImage(img, 0, 0);
      resolve(canvas.toDataURL('image/png'));
    };
    img.onerror = (error) => reject(error);
    img.src = url;
  });
}

const payNow = async () => {
  const res = await fetch('https://ganpati-backend.onrender.com/create-order', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ ...form.value, amount: form.value.amount * 100 }) // send in paise
  })

  const { order, userData } = await res.json()

  const options = {
    key: 'rzp_test_t7gGMihZiizmQm',
    amount: order.amount,
    currency: 'INR',
    name: 'Rishi Sadashiva Devadiga',
    order_id: order.id,
    handler: async (response) => {
      const res = await fetch('https://ganpati-backend.onrender.com/verify-payment', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          ...userData,
          razorpay_order_id: response.razorpay_order_id,
          razorpay_payment_id: response.razorpay_payment_id,
          razorpay_signature: response.razorpay_signature
        })
      });
      const result = await res.json()
      if (result.status === 'success') {
        alert('Payment successful')
        data.value.status = 'success';
        generateReceipt({
          ...userData,
          transaction_type: 'online',
          amount: userData.amount, // This is in paise, adjust if needed
          razorpay_order_id: response.razorpay_order_id,
          razorpay_payment_id: response.razorpay_payment_id
        });
      } else {
        alert('Payment failed')
        data.value.status = 'failure';
      }
    }
  }

  new Razorpay(options).open()
}

const cashPay = async () => {
  const res = await fetch('https://ganpati-backend.onrender.com/cash-payment', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(form.value)
  });
  const result = await res.json();
  if (result.status === 'success') {
    alert('Cash payment recorded');
    data.value.status = 'success';
    generateReceipt({
      ...form.value,
      transaction_type: 'cash',
      ...(transaction_type.value === 'cash' ? { status: status.value } : {}),
      amount: form.value.amount * 100, // convert to paise for consistency with PDF
      razorpay_order_id: 'CASH',
      razorpay_payment_id: 'CASH'
    });
  } else {
    alert('Cash payment failed');
    data.value.status = 'Failure';
  }
}
</script>


<style>
body {
  background: url(https://i.pinimg.com/736x/16/ef/14/16ef14a357bb8ced9cb4caa5a7ca4bb9.jpg) no-repeat center center fixed;
  background-size: cover;
}
.card {
  border-radius: 1rem;
}

.loading-overlay {
  position: fixed;
  top: 0; left: 0; right: 0; bottom: 0;
  background: rgba(255,255,255,0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
}
</style>