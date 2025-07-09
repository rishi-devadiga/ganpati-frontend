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
              <div v-if="form.transaction_type === 'cash'" class="mb-3">
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
import { toWords } from 'number-to-words'

const form = ref({
  name: '',
  address: '',
  phone: '',
  email: '',
  transaction_type: '',
  amount: 0,
  status: '' // 1. Add status field to form
})
const data = ref({ status: '' }) // 1. Track payment status
const loading = ref(false) // 2. Track loading state


async function generateReceipt(data) {
  const doc = new jsPDF({orientation: 'landscape', unit: 'mm'}); // A5 size
  loading.value = true; // 3. Set loading state to true

  const bgUrl = '/receipt.jpg'; // path to your background image
  const bgData = await loadImageAsBase64(bgUrl);

  const pageWidth = doc.internal.pageSize.getWidth();
  const pageHeight = doc.internal.pageSize.getHeight();
  doc.addImage(bgData, 'JPEG', 0, 0, pageWidth, pageHeight);

  // Adjust size and position as needed
  const amountInRupees = data.amount / 100;
  const amountInWords = toWords(amountInRupees) + ' rupees only';

  doc.setFontSize(16);
  doc.setTextColor(0, 0, 0);
  doc.text(`${data.name} ${data.address}`, 150, 85);
  doc.text(`${data.transaction_type}`, 50, 120);
  doc.text(`${(data.amount / 100).toFixed(2)}`, 70, 150);
  doc.text(`${data.id}`, 60, 70);
  doc.text(`${new Date().toLocaleString()}`, 230, 70);
  doc.text(`${amountInWords}`, 110, 100);


  const pdfBlob = doc.output("blob");
  console.log("PDF size (MB):", pdfBlob.size / (1024 * 1024));

  const formData = new FormData();
  formData.append("email", form.value.email);
  formData.append("pdf", pdfBlob, "receipt.pdf");

  try {
    await axios.post("https://ganpati-backend.onrender.com/send-receipt", formData, {
      headers: { "Content-Type": "multipart/form-data" },
    });
    alert("Receipt sent successfully!");
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
      resolve(canvas.toDataURL('image/jpeg', 0.1)); // Adjust quality as needed
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
          id: result.id, // Use the ID from the backend
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
      id: result.id, // Use the ID from the backend
      transaction_type: 'cash',
      ...(form.value.transaction_type === 'cash' ? { status: form.value.status } : {}),
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