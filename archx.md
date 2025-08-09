# **Product Requirements Document (PRD)**  
**Product Name:** ArchX – Design Studio  
**Version:** 1.0  
**Owner:** Product & Engineering Team  

---

## **1. Overview**
ArchX – Design Studio is a web platform that allows users to design and customize interior spaces with AI-generated visual outputs. Users can start projects, select rooms and themes, upload reference images, and apply material changes (e.g., Sunmica) interactively. The platform will store all projects, images, and generations for future reference.

**Platform Stack:**
- **Frontend:** Next.js  
- **Backend:** FastAPI (Dockerized)  
- **Database:** MySQL  
- **Storage:** AWS S3  
- **AI Service:** OpenAI Image Generation API  

---

## **2. Objectives**
- Provide an intuitive and modern interface for interior design customization.  
- Enable AI-powered visual transformations of uploaded or sample images.  
- Store and manage user projects with secure cloud storage.  
- Ensure fast and responsive experience across devices.  

---

## **3. Core Features**

### **3.1 User Authentication**
- **Login Page**
  - Input: Phone Number (India format `+91` prefix, 10-digit validation).
  - **Database Table:** `users`
    ```sql
    users (
      id INT PRIMARY KEY AUTO_INCREMENT,
      phone_number VARCHAR(15) NOT NULL,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
    );
    ```
  - OTP-based verification (future enhancement).

---

### **3.2 Dashboard**
- **User Actions**
  1. **Start a New Project**
  2. **Room Selection** – Bedroom / Hall / Kitchen
  3. **Theme Selection** – Modern / Minimal / Legacy
  4. **Manual Instructions** – Free text input for AI customization notes.
  5. **Image Upload / Capture** – Upload from device or capture via camera.
     - Show sample images for each room & theme.

- **Storage Schema**
  - Uploaded images stored in **S3**:
    ```
    s3://archx/user_{id}/projects/{project_id}/images/{filename}
    ```

- **Database Table:** `user_projects`
    ```sql
    user_projects (
      id INT PRIMARY KEY AUTO_INCREMENT,
      user_id INT NOT NULL,
      room_type ENUM('bedroom','hall','kitchen'),
      theme ENUM('modern','minimal','legacy'),
      instructions TEXT,
      image_url VARCHAR(255),
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
    );
    ```

---

### **3.3 Edit Image Page**
- **Material Store Table:** Predefined materials (e.g., 10 Sunmica samples).
- **User Interaction**
  - Select Sunmica sample.
  - Apply to image via OpenAI image editing API.

- **AI Processing Flow**
  1. Take original uploaded image.
  2. Replace Sunmica areas with selected sample texture.
  3. Render new image for preview.

- **Output Storage**
  - Generated images in S3:
    ```
    s3://archx/user_{id}/projects/{project_id}/generations/{filename}
    ```

- **Database Table:** `user_project_generations`
    ```sql
    user_project_generations (
      id INT PRIMARY KEY AUTO_INCREMENT,
      project_id INT NOT NULL,
      generation_url VARCHAR(255),
      selected_material VARCHAR(100),
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    ```

---

### **3.4 View Page**
- **Functionality**
  - Display original confirmed image.
  - Display generated image side-by-side.
- **Data Retrieval**
  - Fetch from `user_project_generations` by `user_id`.

---

## **4. Non-Functional Requirements**
- **Performance:** Image generation < 15 seconds for standard edits.  
- **Security:** Signed URLs for S3 access; authenticated API calls.  
- **Scalability:** Support 1,000+ concurrent users.  
- **Availability:** 99.9% uptime target.  

---

## **5. Future Enhancements**
- OTP-based login.  
- Component-level editing (handles, panels, etc.).  
- Multi-theme render previews.  
- AR visualization for mobile.  

---

## **6. System Architecture**
1. **Frontend (Next.js)** – Responsive UI, API calls to FastAPI.  
2. **Backend (FastAPI + Docker)** – Auth, image orchestration, DB/S3 integration.  
3. **Database (MySQL)** – Store user, project, and generation data.  
4. **Storage (S3)** – Store original uploads and generated outputs.  
5. **AI Layer (OpenAI)** – Handle image processing/editing.  

---
