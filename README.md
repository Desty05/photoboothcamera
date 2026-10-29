# photoboothcamera
import streamlit as st
import cv2
import numpy as np
from PIL import Image, ImageDraw, ImageFont
import io
import datetime

# Page configuration
st.set_page_config(page_title="Photobooth Pesona Palopo AI", page_icon="📸", layout="centered")

st.title("📸 Photobooth Pesona Palopo AI")
st.caption("Aplikasi Photobooth Interaktif Wisata & Kebudayaan Palopo")

# Sidebar settings
st.sidebar.header("⚙️ Pengaturan Photobooth")

frame_option = st.sidebar.selectbox(
    "🏛️ Pilih Bingkai Wisata Palopo",
    ["Istana Kedatuan Luwu", "Masjid Jami Tua Palopo", "Air Terjun Latuppa", "Pantai Labombo"]
)

filter_option = st.sidebar.selectbox(
    "🎨 Pilih Filter AI Simulation",
    ["Normal / Asli", "AI Anime / Soft Glow", "AI Oil Painting", "AI Cyberpunk Neon", "AI Vintage Comic", "AI Watercolor"]
)

# Session state for captured photos
if 'photos' not in st.session_state:
    st.session_state.photos = []

def apply_ai_filter(img_np, filter_type):
    """Applies artistic AI simulation filters using OpenCV"""
    img = img_np.copy()
    if filter_type == "AI Anime / Soft Glow":
        # Smooth and brighten
        smooth = cv2.bilateralFilter(img, 9, 75, 75)
        hsv = cv2.cvtColor(smooth, cv2.COLOR_RGB2HSV)
        hsv[:, :, 1] = cv2.multiply(hsv[:, :, 1], 1.2)
        return cv2.cvtColor(hsv, cv2.COLOR_HSV2RGB)
    
    elif filter_type == "AI Oil Painting":
        # Stylization / Oil paint effect
        return cv2.stylization(img, sigma_s=60, sigma_r=0.45)
    
    elif filter_type == "AI Cyberpunk Neon":
        # Neon cyan and magenta boost
        img_float = img.astype(np.float32)
        img_float[:, :, 0] = np.clip(img_float[:, :, 0] * 1.3 + 30, 0, 255) # Red/Magenta
        img_float[:, :, 2] = np.clip(img_float[:, :, 2] * 1.4 + 40, 0, 255) # Blue/Cyan
        return img_float.astype(np.uint8)
    
    elif filter_type == "AI Vintage Comic":
        # Pencil sketch / comic outline
        gray = cv2.cvtColor(img, cv2.COLOR_RGB2GRAY)
        blur = cv2.medianBlur(gray, 5)
        edges = cv2.adaptiveThreshold(blur, 255, cv2.ADAPTIVE_THRESH_MEAN_C, cv2.THRESH_BINARY, 9, 9)
        color = cv2.bilateralFilter(img, 9, 250, 250)
        return cv2.bitwise_and(color, color, mask=edges)
    
    elif filter_type == "AI Watercolor":
        # Watercolor effect
        res, _ = cv2.pencilSketch(img, sigma_s=60, sigma_r=0.07, shade_factor=0.05)
        return res
    
    return img

def add_palopo_frame(img_np, location_name):
    """Adds Palopo regional frame overlay"""
    h, w, _ = img_np.shape
    pil_img = Image.fromarray(img_np)
    draw = ImageDraw.Draw(pil_img)
    
    # Gold border
    border_width = 12
    draw.rectangle([0, 0, w, h], outline="#EAB308", width=border_width)
    
    # Banner at bottom
    banner_height = 40
    draw.rectangle([border_width, h - banner_height - border_width, w - border_width, h - border_width], fill=(15, 23, 42, 200))
    
    # Text overlay
    text = f"📍 {location_name} - Palopo"
    draw.text((w // 2, h - 25), text, fill="#FACC15", anchor="mm")
    
    return np.array(pil_img)

def generate_photo_strip(photos_list):
    """Generates 3-photo vertical strip with branding"""
    strip_w = 400
    photo_w = 350
    photo_h = 262
    gap = 18
    header_h = 75
    footer_h = 60
    
    total_h = header_h + 3 * photo_h + 2 * gap + footer_h
    
    strip_img = Image.new("RGB", (strip_w, total_h), "#FFFFFF")
    draw = ImageDraw.Draw(strip_img)
    
    # Header
    draw.text((strip_w // 2, 28), "PESONA PALOPO AI", fill="#0F172A", anchor="mm")
    draw.text((strip_w // 2, 50), "Wisata & Kebudayaan Kota Palopo", fill="#64748B", anchor="mm")
    
    # Draw photos
    for idx, p_np in enumerate(photos_list):
        p_pil = Image.fromarray(p_np).resize((photo_w, photo_h))
        x_pos = (strip_w - photo_w) // 2
        y_pos = header_h + idx * (photo_h + gap)
        strip_img.paste(p_pil, (x_pos, y_pos))
    
    # Footer
    footer_y = header_h + 3 * photo_h + 2 * gap + 20
    date_str = datetime.datetime.now().strftime("%d %b %Y")
    draw.text((strip_w // 2, footer_y), "✨ Luwu Heritage Photobooth ✨", fill="#0F172A", anchor="mm")
    draw.text((strip_w // 2, footer_y + 20), f"Diproses dengan AI Filter • {date_str}", fill="#94a3b8", anchor="mm")
    
    return strip_img

# Main Camera Section
st.subheader("📷 Ambil Foto")
camera_photo = st.camera_input("Klik tombol di bawah untuk mengambil foto:")

if camera_photo is not None:
    # Convert buffer to image
    image = Image.open(camera_photo)
    img_np = np.array(image)
    
    # Apply filter and frame
    filtered_img = apply_ai_filter(img_np, filter_option)
    framed_img = add_palopo_frame(filtered_img, frame_option)
    
    col1, col2 = st.columns(2)
    with col1:
        st.image(framed_img, caption="Hasil Foto saat ini", use_column_width=True)
    
    with col2:
        if st.button("➕ Tambahkan ke Photo Strip", use_container_width=True):
            if len(st.session_state.photos) < 3:
                st.session_state.photos.append(framed_img)
                st.success(f"Foto berhasil ditambahkan! ({len(st.session_state.photos)}/3)")
            else:
                st.warning("Sudah ada 3 foto! Silakan reset jika ingin mengambil ulang.")

st.divider()

# Photo Strip Section
st.subheader("🖼️ Preview Photo Strip (3 Foto)")

if len(st.session_state.photos) > 0:
    st.info(f"Jumlah foto saat ini: {len(st.session_state.photos)}/3")
    
    # If 3 photos collected, render strip
    if len(st.session_state.photos) == 3:
        strip_result = generate_photo_strip(st.session_state.photos)
        
        st.image(strip_result, caption="Hasil Photobooth Strip Ready to Print/Download", width=320)
        
        # Download button
        buf = io.BytesIO()
        strip_result.save(buf, format="PNG")
        byte_im = buf.getvalue()
        
        st.download_button(
            label="📥 Download Photo Strip (PNG)",
            data=byte_im,
            file_name=f"photobooth-palopo-{datetime.datetime.now().strftime('%Y%m%d_%H%M%S')}.png",
            mime="image/png",
            use_container_width=True
        )

# Reset Button
if st.button("🔄 Reset Sesi Foto", use_container_width=True):
    st.session_state.photos = []
    st.rerun()
