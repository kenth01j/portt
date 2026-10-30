# Dark Personal Portfolio

## Project structure

```text
my-portfolio/
├── .gitignore
├── package-lock.json
├── package.json
├── README.md
├── render.yaml
├── backend/
│   ├── controllers/
│   │   ├── authController.js
│   │   └── portfolioController.js
│   ├── middleware/
│   │   └── verifyToken.js
│   ├── models/
│   │   ├── Admin.js
│   │   └── PortfolioData.js
│   ├── routes/
│   │   ├── auth.js
│   │   ├── portfolio.js
│   │   └── uploads.js
│   ├── uploads/              # uploaded images are stored locally
│   ├── .env                  # create locally; do not commit
│   ├── .env.example
│   ├── package-lock.json
│   ├── package.json
│   └── server.js
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── ProtectedRoute.jsx
│   │   │   └── Sidebar.jsx
│   │   ├── context/
│   │   │   └── PortfolioContext.jsx
│   │   ├── pages/
│   │   │   ├── AdminDashboard.jsx
│   │   │   ├── AdminLogin.jsx
│   │   │   ├── Home.jsx
│   │   │   └── Resume.jsx
│   │   ├── utils/
│   │   │   └── api.js
│   │   ├── App.jsx
│   │   ├── index.css
│   │   └── main.jsx
│   ├── index.html
│   ├── package-lock.json
│   ├── package.json
│   ├── postcss.config.js
│   ├── tailwind.config.js
│   ├── vercel.json
│   └── vite.config.js
```

## Local setup

Prerequisites: Node.js 20+ and a running MongoDB instance (local or hosted).

1. Install the root runner and both app dependencies:

   ```powershell
   npm install
   npm --prefix backend install
   npm --prefix frontend install
   ```

2. Create `backend/.env` from `backend/.env.example`. For local MongoDB, the
   example URI is ready to use. Replace `JWT_SECRET` with a long, unique random
   value, and adjust `FRONTEND_URL` if the frontend uses a different origin.

   ```dotenv
   PORT=5000
   MONGODB_URI=mongodb://127.0.0.1:27017/personal-portfolio
   JWT_SECRET=use-a-long-unique-random-secret
   FRONTEND_URL=http://localhost:5173
   ```

   Optionally, create `frontend/.env` and set
   `VITE_API_URL=http://localhost:5000/api`. The API wrapper uses that value;
   this is also its default.

3. Start both applications from the project root:

   ```powershell
   npm run dev
   ```

   Or open two terminals and run `npm --prefix backend run dev` and
   `npm --prefix frontend run dev` separately. The backend is at
   `http://localhost:5000`; the Vite frontend is at `http://localhost:5173`.

4. Create the initial admin exactly once using Postman:

   - Method: `POST`
   - URL: `http://localhost:5000/api/auth/register`
   - Body: **raw** → **JSON**
   - Example:

     ```json
     {
       "username": "portfolio-admin",
       "password": "choose-a-unique-password-at-least-12-characters"
     }
     ```

   Registration is deliberately available only as a one-time bootstrap: the
   backend reserves a fixed admin document ID, so subsequent registration
   attempts are rejected. Keep the password and JWT secret private. Then visit
   `http://localhost:5173/admin` and sign in. Portfolio editing is available at
   `/admin/dashboard`.

   In the dashboard, choose multiple profile photos and select which one is the
   main photo before saving. Visitors see the main photo on the profile card; the
   other saved photos are revealed in the dropdown gallery after they open that
   photo.

## Deploying online

The site uses a frontend, API, database, and uploaded image files. Deploy them
as separate services: Vercel for the Vite frontend, a Node host with persistent
storage for the Express backend, and MongoDB Atlas for the database. Do not use
the local MongoDB address or temporary server storage for a live deployment.

### 1. Create a MongoDB Atlas database

1. Create an Atlas project and cluster, then create a database user with a
   strong, unique password.
2. In **Network Access**, allow connections from your backend host. If that
   host uses dynamic outbound IPs, its documented outbound IP ranges may be
   needed; only use `0.0.0.0/0` if necessary, with a strong database password.
3. Copy the Atlas connection string and replace the username/password
   placeholders. Use the `Portfolio` database name, for example:
   `mongodb+srv://USER:PASSWORD@YOUR-CLUSTER.mongodb.net/Portfolio`.
4. Migrate local data from Compass if you want to keep it. Export/import the
   `admins` and `portfoliodatas` collections into Atlas. Never put database
   credentials in frontend environment variables or commit them.

### 2. Deploy the backend with persistent image storage

Use Render for the Express backend. The root `render.yaml` defines the Node
service and a persistent disk for uploaded images. Render persistent disks may
require a paid instance plan; an ephemeral filesystem will lose uploaded photos
on restart or redeploy.

1. In Render, create a **Blueprint** from this repository and select the root
   `render.yaml`. Alternatively, create a Web Service manually with root
   directory `backend`, build command `npm install`, start command `npm start`,
   and a persistent disk mounted at `/var/data`.
2. Set these backend environment variables in the Render dashboard:

   ```dotenv
   MONGODB_URI=<your Atlas connection string>
   JWT_SECRET=<a new long random secret>
   FRONTEND_URL=https://<your-vercel-domain>
   BACKEND_PUBLIC_URL=https://<your-backend-domain>
   UPLOAD_DIR=/var/data/portfolio-uploads
   ```

   Set `PORT` only if your backend host requires a fixed value; otherwise the
   service reads the host-provided port. Add any custom frontend domain to
   `FRONTEND_URL` as a comma-separated origin.

3. Wait for the backend deployment and verify
   `https://<your-backend-domain>/api/health` returns `{"status":"ok"}`.

### 3. Deploy the frontend to Vercel

1. Push the project to a Git provider such as GitHub, then import that repository
   into Vercel. This workspace is not currently a Git repository, so it must be
   initialized and pushed before Vercel can import it.
2. In Vercel project settings, set **Root Directory** to `frontend`. Vercel
   should detect Vite; use `npm run build` and `dist` if prompted.
3. Add this frontend environment variable in Vercel and set it for the
   Production environment:

   ```dotenv
   VITE_API_URL=https://<your-backend-domain>/api
   ```

4. Deploy the frontend, then copy its final HTTPS domain into the backend's
   `FRONTEND_URL` setting and redeploy/restart the backend.
5. Visit the Vercel URL and test the public page, admin login, and an image
   upload. The `frontend/vercel.json` rewrite lets client-side routes such as
   `/admin` and `/resume` load directly.

### 4. Preserve local data and uploaded images

The existing local MongoDB database is not automatically copied to Atlas.
Export/import its `admins` and `portfoliodatas` collections if you need that
content. Copy files from `backend/uploads/` to the backend's persistent upload
directory, keeping the filenames; old `localhost:5000` image URLs in portfolio
data are automatically served from the configured backend origin in production.
Keep local admin credentials private and verify login against the migrated
database before removing any local data.

## API overview

- `GET /api/portfolio` — public portfolio data (creates an empty default record
  on first read).
- `PUT /api/portfolio` — update the hero, skills, project/certificate lists, or
  ATS resume text; requires a valid JWT in the `Authorization: Bearer` header.
- `POST /api/portfolio/projects` and `DELETE /api/portfolio/projects/:id` —
  project management; requires a token.
- `POST /api/portfolio/certificates` and
  `DELETE /api/portfolio/certificates/:id` — certificate management; requires
  a token.
- `POST /api/uploads` — upload a JPEG, PNG, WebP, or GIF image (maximum 5 MB)
  using multipart form field `image`; requires a token. The dashboard lets you
  pick profile, project, and certificate images directly from your device.
  Multiple profile images can be uploaded, with one selected as the public
  profile photo; the others appear in the photo gallery opened from that image.
- `POST /api/auth/login` — returns a 12-hour JWT.

Uploaded images are served from `/uploads` on the backend and stored in
`backend/uploads/`. Back up this folder along with your database when deploying
or moving the portfolio to another machine.
