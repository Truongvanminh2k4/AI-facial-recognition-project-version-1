<!DOCTYPE html>
<html lang="vi">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Đăng ký thông tin sinh viên + Chụp ảnh</title>
  <style>
    body { font-family: Arial, sans-serif; background: #f5f5f5; margin:0; padding:20px; }
    .form-container { max-width: 420px; margin:auto; background:#fff; padding:20px; border-radius:10px; box-shadow:0 2px 8px rgba(0,0,0,0.1); }
    h2 { text-align:center; margin-bottom:15px; }
    label { display:block; margin-top:10px; font-weight:bold; }
    input, select, button { width:100%; padding:8px; margin-top:5px; border:1px solid #ccc; border-radius:5px; box-sizing:border-box; }
    button { cursor:pointer; background:#4CAF50; color:white; border:none; padding:10px; }
    button:hover { background:#45a049; }
    .camera-area { margin-top:15px; display:flex; gap:10px; flex-direction:column; align-items:center; }
    video { width:100%; max-height:300px; border-radius:8px; background:#000; }
    canvas { display:none; }
    .controls { width:100%; display:flex; gap:8px; }
    .controls button, .controls select { flex:1; }
    .preview { width:100%; margin-top:10px; text-align:center; }
    .preview img { max-width:100%; border-radius:8px; border:1px solid #ddd; }
    .note { font-size:0.9rem; color:#555; margin-top:6px; }
    .small { font-size:0.85rem; color:#666; margin-top:6px; text-align:center; }
  </style>
</head>
<body>
  <div class="form-container">
    <h2>Đăng ký thông tin sinh viên</h2>
    <form id="studentForm">
      <label for="name">Tên:</label>
      <input type="text" id="name" name="name" required>

      <label for="mssv">MSSV:</label>
      <input type="text" id="mssv" name="mssv" required>

      <label for="class">Lớp:</label>
      <input type="text" id="class" name="class" required>

      <label for="subject">Môn học:</label>
      <select id="subject" name="subject" required>
        <option value="">--Chọn môn--</option>
        <option>IOT</option>
      </select>

      <label for="day">Thứ (2 - 8):</label>
      <select id="day" name="day" required>
        <option value="">--Chọn thứ--</option>
        <option value="2">2</option>
        <option value="3">3</option>
        <option value="4">4</option>
        <option value="5">5</option>
        <option value="6">6</option>
        <option value="7">7</option>
        <option value="8">8</option>
      </select>

      <label for="shift">Tiết:</label>
      <select id="shift" name="shift" required>
        <option value="">--Chọn tiết--</option>
        <option>1</option>
        <option>4</option>
        <option>7</option>
        <option>10</option>
      </select>

      <!-- Camera / Capture -->
      <div class="camera-area" aria-live="polite">
        <label>Ảnh sinh viên (chụp trực tiếp hoặc tải lên):</label>

        <!-- Video stream nếu hỗ trợ -->
        <video id="video" autoplay playsinline></video>

        <!-- Fallback cho mobile/thiết bị không hỗ trợ getUserMedia -->
        <input id="fileInput" type="file" accept="image/*" capture="environment" style="display:none" />

        <div class="controls">
          <select id="facingMode" title="Chọn camera (nếu trình duyệt hỗ trợ)">
            <option value="environment">Camera sau</option>
            <option value="user">Camera trước</option>
          </select>
          <button type="button" id="startCamera">Bật Camera</button>
          <button type="button" id="takePhoto">Chụp</button>
        </div>

        <div class="small">Nếu trình duyệt/thiết bị không cho phép camera, bấm vào "Tải ảnh từ thiết bị" để chọn ảnh từ máy.</div>
        <div style="display:flex;gap:8px;margin-top:8px;">
          <button type="button" id="pickFileBtn">Tải ảnh từ thiết bị</button>
          <button type="button" id="downloadPhotoBtn" disabled>Tải JPG về máy</button>
        </div>

        <div class="preview" id="preview">
          <!-- Ảnh xem trước sẽ hiển thị ở đây -->
        </div>
        <canvas id="canvas"></canvas>
      </div>

      <button type="submit" id="submitBtn">Gửi thông tin</button>
    </form>
  </div>

  <!-- Firebase SDK (giữ nguyên nếu bạn vẫn muốn gửi dữ liệu lên Firebase) -->
  <script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/12.4.0/firebase-app.js";
    import { getDatabase, ref, push, set } from "https://www.gstatic.com/firebasejs/12.4.0/firebase-database.js";
    import { getAnalytics } from "https://www.gstatic.com/firebasejs/12.4.0/firebase-analytics.js";

    const firebaseConfig = {
      apiKey: "AIzaSyA0vOnUVTTx8ciKYWPqbcucOAYveA6hRww",
      authDomain: "demo3-1b7e0.firebaseapp.com",
      databaseURL: "https://demo3-1b7e0-default-rtdb.firebaseio.com",
      projectId: "demo3-1b7e0",
      storageBucket: "demo3-1b7e0.firebasestorage.app",
      messagingSenderId: "623845481855",
      appId: "1:623845481855:web:d7b884b0681f054467ce4a",
      measurementId: "G-V6BH2FNYGZ"
    };

    const app = initializeApp(firebaseConfig);
    const analytics = getAnalytics(app);
    const db = getDatabase(app);

    // --- Camera logic ---
    const video = document.getElementById("video");
    const canvas = document.getElementById("canvas");
    const preview = document.getElementById("preview");
    const startCameraBtn = document.getElementById("startCamera");
    const takePhotoBtn = document.getElementById("takePhoto");
    const pickFileBtn = document.getElementById("pickFileBtn");
    const fileInput = document.getElementById("fileInput");
    const downloadBtn = document.getElementById("downloadPhotoBtn");
    const facingModeSelect = document.getElementById("facingMode");

    let stream = null;
    let lastImageBlob = null;

    async function startCamera() {
      // Hủy stream cũ nếu có
      if (stream) {
        stream.getTracks().forEach(t => t.stop());
        stream = null;
      }

      const facingMode = facingModeSelect.value || "environment";
      const constraints = {
        video: { facingMode: { ideal: facingMode } },
        audio: false
      };

      try {
        stream = await navigator.mediaDevices.getUserMedia(constraints);
        video.srcObject = stream;
        video.style.display = "block";
      } catch (err) {
        console.warn("Không thể bật camera:", err);
        alert("Trình duyệt không cho phép hoặc không có camera. Bạn có thể tải ảnh từ thiết bị.");
        // nếu lỗi, hiển thị input file để người dùng chọn ảnh
        fileInput.style.display = "block";
      }
    }

    function dataURLToBlob(dataURL) {
      const parts = dataURL.split(';base64,');
      const mime = parts[0].split(':')[1];
      const bstr = atob(parts[1]);
      let n = bstr.length;
      const u8arr = new Uint8Array(n);
      while (n--) u8arr[n] = bstr.charCodeAt(n);
      return new Blob([u8arr], { type: mime });
    }

    function showPreviewFromBlob(blob) {
      lastImageBlob = blob;
      const url = URL.createObjectURL(blob);
      preview.innerHTML = `<img src="${url}" alt="Preview ảnh">`;
      downloadBtn.disabled = false;
    }

    takePhotoBtn.addEventListener("click", () => {
      // Nếu có stream camera thì chụp từ video, nếu không thì mở file picker
      if (stream) {
        const vw = video.videoWidth;
        const vh = video.videoHeight;
        canvas.width = vw;
        canvas.height = vh;
        const ctx = canvas.getContext("2d");
        // Mirror for front camera? (optional) - not applied here to keep natural photo
        ctx.drawImage(video, 0, 0, vw, vh);
        // Chuyển sang JPG với chất lượng 0.9
        const dataURL = canvas.toDataURL("image/jpeg", 0.9);
        const blob = dataURLToBlob(dataURL);
        showPreviewFromBlob(blob);
      } else {
        // fallback: mở file input
        fileInput.click();
      }
    });

    pickFileBtn.addEventListener("click", () => fileInput.click());

    fileInput.addEventListener("change", (e) => {
      const file = e.target.files && e.target.files[0];
      if (!file) return;
      // Nếu file không phải ảnh, bỏ qua
      if (!file.type.startsWith("image/")) {
        alert("Vui lòng chọn tệp ảnh.");
        return;
      }
      // Convert any image to JPG via canvas to ensure JPG download
      const img = new Image();
      const url = URL.createObjectURL(file);
      img.onload = () => {
        canvas.width = img.naturalWidth;
        canvas.height = img.naturalHeight;
        const ctx = canvas.getContext("2d");
        ctx.drawImage(img, 0, 0);
        const dataURL = canvas.toDataURL("image/jpeg", 0.9);
        const blob = dataURLToBlob(dataURL);
        showPreviewFromBlob(blob);
        URL.revokeObjectURL(url);
      };
      img.src = url;
    });

    downloadBtn.addEventListener("click", () => {
      if (!lastImageBlob) return;
      const mssv = document.getElementById("mssv").value || "unknown";
      const timestamp = new Date().toISOString().replace(/[:.]/g, "-");
      const filename = `photo_${mssv}_${timestamp}.jpg`;
      const link = document.createElement("a");
      link.href = URL.createObjectURL(lastImageBlob);
      link.download = filename;
      document.body.appendChild(link);
      link.click();
      link.remove();
      // release object url after a short time
      setTimeout(() => URL.revokeObjectURL(link.href), 10000);
    });

    // Tắt camera khi rời trang
    window.addEventListener("beforeunload", () => {
      if (stream) stream.getTracks().forEach(t => t.stop());
    });

    startCameraBtn.addEventListener("click", startCamera);

    // --- Form submit xử lý: lưu vào Firebase + gắn link ảnh (nếu cần) ---
    const form = document.getElementById("studentForm");
    form.addEventListener("submit", async (e) => {
      e.preventDefault();

      const nameValue = document.getElementById("name").value.toUpperCase();
      const classValue = document.getElementById("class").value.toUpperCase();
      const mssvValue = document.getElementById("mssv").value;
      const subjectValue = document.getElementById("subject").value;
      const thuValue = document.getElementById("day").value;
      const tietValue = document.getElementById("shift").value;
      const thoigian = new Date().toLocaleString("vi-VN");

      // Tạo dữ liệu text
      const data = {
        ten: nameValue,
        mssv: mssvValue,
        lop: classValue,
        mon: subjectValue,
        thu: thuValue,
        tiet: tietValue,
        thoigian: thoigian
      };

      try {
        // Lưu dữ liệu lên Realtime Database
        const newRef = push(ref(db, "DangKySinhVien"));
        await set(newRef, data);

        // Nếu có ảnh (lastImageBlob), tự động trigger download (đã có nút), 
        // nếu muốn, ta cũng có thể upload ảnh lên Firebase Storage (không có trong code này).
        if (lastImageBlob) {
          // TỰ ĐỘNG tải về (để chắc chắn người dùng có file): sẽ gọi same logic như nút download
          const timestamp = new Date().toISOString().replace(/[:.]/g, "-");
          const filename = `photo_${mssvValue || "unknown"}_${timestamp}.jpg`;
          const link = document.createElement("a");
          link.href = URL.createObjectURL(lastImageBlob);
          link.download = filename;
          document.body.appendChild(link);
          link.click();
          link.remove();
          setTimeout(() => URL.revokeObjectURL(link.href), 10000);

          // === Nếu bạn muốn **upload ảnh lên Firebase Storage**, mở phần bên dưới và cấu hình Storage.
          // LƯU Ý: Firebase Storage cần import thêm module firebase/storage và thay đổi security rules.
          /*
          import { getStorage, ref as sref, uploadBytes, getDownloadURL } from "https://www.gstatic.com/firebasejs/12.4.0/firebase-storage.js";
          const storage = getStorage(app);
          const storageRef = sref(storage, `student_photos/${filename}`);
          await uploadBytes(storageRef, lastImageBlob);
          const downloadURL = await getDownloadURL(storageRef);
          // bạn có thể thêm downloadURL vào object data và cập nhật DB
          await set(newRef, {...data, photoURL: downloadURL});
          */
        }

        alert("✅ Đăng ký thành công! Ảnh đã được tải về máy (nếu có).");
        form.reset();
        preview.innerHTML = "";
        lastImageBlob = null;
        downloadBtn.disabled = true;
      } catch (err) {
        console.error("Lỗi lưu Firebase:", err);
        alert("❌ Có lỗi xảy ra khi lưu dữ liệu!");
      }
    });

    // Tự động thử bật camera nhẹ nhàng nếu người dùng cho phép (không bắt buộc)
    (async function tryAutoStart() {
      if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) return;
      // Không tự bật nếu trang mới load — chờ user bấm "Bật Camera" (lý do quyền UX/ bảo mật).
    })();
  </script>
</body>
</html>
