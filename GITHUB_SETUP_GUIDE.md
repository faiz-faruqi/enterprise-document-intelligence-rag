# GitHub Setup Guide for First-Time Users

This guide will walk you through setting up your GitHub repository from scratch and publishing your Enterprise RAG Platform architecture portfolio.

---

## Prerequisites

- A computer with internet access
- Email address for GitHub account
- The `enterprise-rag-platform` folder you've downloaded

---

## Step 1: Create a GitHub Account

1. Go to https://github.com
2. Click **Sign up** in the top right
3. Enter your email address (use professional email, e.g., yourname@gmail.com)
4. Create a password (strong password recommended)
5. Choose a username (professional, e.g., `john-smith-architect` or `jsmith-ai`)
   - This appears in your portfolio URL: `github.com/your-username`
   - Keep it professional — this is part of your brand
6. Verify you're human (solve the puzzle)
7. Click **Create account**
8. Check your email for verification code
9. Enter the code when prompted
10. Skip personalization questions (or answer if you prefer)

---

## Step 2: Install Git on Your Computer

### Windows
1. Download Git from https://git-scm.com/download/win
2. Run the installer
3. Use default settings (click Next through the wizard)
4. Open **Git Bash** from Start Menu after installation

### Mac
1. Open **Terminal** (Cmd + Space, type "Terminal")
2. Type: `git --version`
3. If Git is not installed, Mac OS will prompt you to install it
4. Follow the prompts

### Linux
```bash
# Ubuntu/Debian
sudo apt-get install git

# Fedora
sudo dnf install git
```

---

## Step 3: Configure Git (One-Time Setup)

Open Terminal (Mac/Linux) or Git Bash (Windows) and run:

```bash
git config --global user.name "Your Full Name"
git config --global user.email "your.email@example.com"
```

Replace with your actual name and email.

**Example**:
```bash
git config --global user.name "John Smith"
git config --global user.email "john.smith@gmail.com"
```

---

## Step 4: Create a New Repository on GitHub

1. Log in to GitHub (https://github.com)
2. Click the **+** icon in the top right corner
3. Select **New repository**
4. Fill in the details:
   - **Repository name**: `enterprise-document-intelligence-rag`
   - **Description**: `Reference architecture for enterprise document intelligence using RAG`
   - **Visibility**: Choose **Public** (for portfolio visibility)
   - **DO NOT** check "Initialize with README" (we already have one)
5. Click **Create repository**

You'll see a page with setup instructions — **keep this page open**, we'll use it next.

---

## Step 5: Upload Your Files to GitHub

### Option A: Using GitHub Web Interface (Easiest for First-Timers)

1. On your new repository page, click **uploading an existing file**
2. Drag the entire `enterprise-rag-platform` folder into the upload area
   - **OR** click **choose your files** and select all files
3. Scroll down and click **Commit changes**

**Done!** Your repository is now live.

### Option B: Using Command Line (Recommended for Architects)

This is the professional way that gives you full control.

#### 5.1: Navigate to Your Folder

Open Terminal (Mac/Linux) or Git Bash (Windows):

```bash
cd /path/to/enterprise-rag-platform
```

**Example (Mac)**:
```bash
cd ~/Downloads/enterprise-rag-platform
```

**Example (Windows)**:
```bash
cd C:/Users/YourName/Downloads/enterprise-rag-platform
```

**Tip**: Type `cd ` (with a space), then drag the folder into Terminal — it will auto-fill the path.

#### 5.2: Initialize Git Repository

```bash
git init
```

You'll see: `Initialized empty Git repository`

#### 5.3: Add All Files

```bash
git add .
```

This stages all files for commit. The `.` means "everything in this folder."

#### 5.4: Create Your First Commit

```bash
git commit -m "Initial commit: Enterprise RAG Platform architecture"
```

#### 5.5: Connect to GitHub

Copy the repository URL from GitHub. It looks like:
```
https://github.com/your-username/enterprise-document-intelligence-rag.git
```

Run:
```bash
git remote add origin https://github.com/your-username/enterprise-document-intelligence-rag.git
```

Replace `your-username` with your actual GitHub username.

#### 5.6: Push to GitHub

```bash
git branch -M main
git push -u origin main
```

**First time pushing?** GitHub will ask you to log in:
- Enter your GitHub username
- For password, use a **Personal Access Token** (not your account password)

**How to create a Personal Access Token**:
1. Go to https://github.com/settings/tokens
2. Click **Generate new token** → **Generate new token (classic)**
3. Name it: `Git Push Token`
4. Set expiration: `90 days` (or longer)
5. Check the box: **repo** (gives full repository access)
6. Click **Generate token**
7. **COPY THE TOKEN** (you won't see it again)
8. Use this token as your password when Git asks

---

## Step 6: Verify Your Repository

1. Go to `https://github.com/your-username/enterprise-document-intelligence-rag`
2. You should see:
   - README.md displayed at the bottom
   - All your folders (architecture, architecture-decision-records, docs, etc.)
   - Professional green checkmarks (files successfully uploaded)

**If you see your files: Congratulations! Your portfolio is live.**

---

## Step 7: Make Your Repository Look Professional

### Add Topics (Tags)

1. On your repository page, click the ⚙️ icon next to "About"
2. Add topics:
   - `rag`
   - `retrieval-augmented-generation`
   - `document-intelligence`
   - `enterprise-architecture`
   - `vector-database`
   - `genai`
   - `llm`
   - `fastapi`
   - `qdrant`
3. Click **Save changes**

### Add a Description

In the same "About" section:
- Description: `Reference architecture for enterprise document intelligence using RAG, n8n, FastAPI, Qdrant, and local embeddings`

---

## Step 8: Update Links in Your README

Your README has placeholder links. Update them:

1. Click **README.md** in your repository
2. Click the **pencil icon** (Edit this file)
3. Find these placeholders and replace:
   - `https://github.com/yourusername/...` → Your actual repository URL
   - `https://linkedin.com/in/yourprofile` → Your LinkedIn URL
   - `https://yourportfolio.com` → Your portfolio site (if you have one)
4. Scroll down and click **Commit changes**

---

## Step 9: Add the Architecture Diagram

You created an architecture diagram earlier. Let's add it:

### Save the Diagram as SVG

1. Go back to the chat where Claude created the diagram
2. Right-click the diagram
3. Select **Save image as...**
4. Save as: `platform-architecture.svg`

### Upload to GitHub

1. In your repository, navigate to `architecture/diagrams/`
2. Click **Add file** → **Upload files**
3. Upload `platform-architecture.svg`
4. Click **Commit changes**

### Update README Reference

The README references `./architecture-diagram.svg`. Update it to:
```markdown
![Platform Architecture](architecture/diagrams/platform-architecture.svg)
```

---

## Step 10: Share Your Portfolio

Your repository is now live at:
```
https://github.com/your-username/enterprise-document-intelligence-rag
```

### Add to LinkedIn

1. Go to your LinkedIn profile
2. Scroll to **Featured** section
3. Click **+** → **Add link**
4. Paste your GitHub repository URL
5. Add title: `Enterprise RAG Platform Architecture`
6. Add description: `Reference architecture demonstrating enterprise GenAI platform patterns for document intelligence`

### Add to Resume/CV

Under "Projects" or "Portfolio":
```
Enterprise Document Intelligence Platform
GitHub: github.com/your-username/enterprise-document-intelligence-rag
Reference architecture for RAG-based document intelligence
```

---

## Common Issues & Solutions

### Issue: "Permission denied (publickey)"

**Solution**: You need to authenticate. Use HTTPS (not SSH) and create a Personal Access Token (see Step 5.6).

### Issue: "Repository not found"

**Solution**: Check you typed the URL correctly. It's case-sensitive.

### Issue: Git says "not a git repository"

**Solution**: Make sure you ran `git init` in the correct folder.

### Issue: Files not showing on GitHub

**Solution**: 
1. Check you committed: `git status` (should say "nothing to commit")
2. Check you pushed: `git log` (should show your commit)
3. Try pushing again: `git push origin main`

---

## Next Steps After GitHub Setup

### 1. Publish Your Blog Post

- Copy the blog post from `docs/blog-post.md`
- Publish on Medium, Dev.to, or your personal blog
- Link to your GitHub repository in the blog post

### 2. Share on LinkedIn

Write a post:
```
I recently architected an Enterprise Document Intelligence platform using RAG.

Instead of building another one-off prototype, I focused on creating a reusable platform pattern with:
- Separation of orchestration, AI services, storage, and delivery
- Vendor-neutral design (swap Qdrant → Pinecone, n8n → Airflow)
- Multi-use-case reusability (same core supports contracts, policies, proposals)

The architecture and decision records are on GitHub: [link]

Key takeaway: enterprises need platforms, not point solutions.

#GenAI #RAG #EnterpriseArchitecture #DocumentIntelligence
```

### 3. Keep Updating

As you enhance the platform:
```bash
# Make changes to files
git add .
git commit -m "Add hybrid search capabilities"
git push origin main
```

---

## Congratulations! 🎉

Your enterprise architecture portfolio is now live on GitHub. This demonstrates:
- ✅ You can design enterprise-grade systems
- ✅ You document architectural decisions (ADRs)
- ✅ You think about trade-offs and vendor neutrality
- ✅ You communicate technical concepts clearly

**This is exactly what hiring managers want to see.**

---

## Questions?

If you run into issues, common fixes:
1. Google the error message (90% of Git errors are solved this way)
2. Check GitHub's docs: https://docs.github.com
3. Ask ChatGPT or Claude for help with specific error messages

**Remember**: Every professional developer has struggled with Git at first. You're doing great! 🚀
