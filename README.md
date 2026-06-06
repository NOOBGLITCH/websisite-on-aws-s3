# Deploy Website to AWS S3 + CloudFront with CI/CD

This guide walks you through setting up a static website on AWS S3 with CloudFront CDN and automatic deployments using GitHub Actions.

## Prerequisites

- An AWS account
- A GitHub repository
- Your static website files (HTML, CSS, JS)

---

## Step 1: Create S3 Bucket

1. Log in to the AWS Management Console and navigate to **S3**.
2. Click **Create bucket**.
3. Enter a globally unique bucket name (e.g., `my-website-bucket`).
4. Select a region (e.g., `ap-south-1`).
5. Keep all other settings as default for now.
6. Click **Create bucket**.

![Create S3 Bucket](https://imgyx.pages.dev/4VeU3)

---

## Step 2: Configure S3 Bucket for Static Website Hosting

1. Open the bucket you just created.
2. Go to the **Properties** tab.
3. Scroll down to **Static website hosting** and click **Edit**.
4. Enable **Static website hosting**.
5. Set the **Index document** to `index.html`.
6. Save the changes.

![Static Website Settings](https://imgyx.pages.dev/4VeU3)

---

## Step 3: Upload Website Files (Initial Upload)

1. Go to the **Objects** tab inside your bucket.
2. Click **Upload**.
3. Drag and drop or select your website files (HTML, CSS, JS).
4. Click **Upload**.

Your site is now live at the S3 website endpoint (shown in the Static website hosting section).

![Upload Files](https://imgyx.pages.dev/rlH0U)

---

## Step 4: Set Up CloudFront CDN

CloudFront speeds up your website by caching content at edge locations worldwide.

### Step 4.1: Create Distribution

1. Navigate to **CloudFront** in the AWS Console.
2. Click **Create distribution**.
3. Under **Origin domain**, select your S3 bucket from the dropdown.
4. Leave the remaining settings as **default** (no changes needed).
5. Scroll down and click **Create distribution**.

![Create Distribution](https://imgyx.pages.dev/OF244)

![Select S3 Origin](https://imgyx.pages.dev/FNTH5)

![Keep default settings](https://imgyx.pages.dev/FNTH5)

### Step 4.2: Update Bucket Policy for CloudFront

CloudFront needs permission to access your S3 bucket.

1. After creating the distribution, go to the **Origins** tab.
2. Click on your S3 origin name.
3. Click **Edit**.
4. Copy the **Origin policy** that appears.

![Copy Origin Policy](https://imgyx.pages.dev/4MhIv)

5. Navigate back to your S3 bucket.
6. Go to **Permissions** → **Bucket policy** → **Edit**.
7. Paste the copied policy and save.

![Edit Bucket Policy](https://imgyx.pages.dev/adFgX)

---

## Step 5: Set Up CI/CD with GitHub Actions

Automate deployments so that every push to your `main` branch updates your website.

### Step 5.1: Create IAM User for Deployment

1. Go to **IAM** in the AWS Console.
2. Click **Users** → **Create user**.
3. Give it a name like `github-actions-deployer`.
4. Select **Attach policies directly**.
5. Attach the following policies:
   - `AmazonS3FullAccess`
   - `CloudFrontFullAccess` (optional, if you want to invalidate cache automatically)
6. Click **Create user**.
7. After creation, go to the **Security credentials** tab.
8. Click **Create access key** and store:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`

### Step 5.2: Add Secrets to GitHub

1. In your GitHub repository, go to **Settings** → **Secrets and variables** → **Actions**.
2. Click **New repository secret** and add:
   - `AWS_ACCESS_KEY_ID` — the key ID from Step 5.1
   - `AWS_SECRET_ACCESS_KEY` — the secret key from Step 5.1
   - `AWS_S3_BUCKET` — your S3 bucket name (e.g., `my-website-bucket`)

### Step 5.3: Add GitHub Actions Workflow

Create a file at `.github/workflows/deploy.yml` in your repository:

```yaml
name: Upload Website to S3 bucket

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      # 1. Log into AWS using the official AWS action
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1

      # 2. Sync using the built-in AWS CLI (stable and explicit)
      - name: Deploy to S3
        run: aws s3 sync . s3://${{ secrets.AWS_S3_BUCKET }} --follow-symlinks --delete
```

---

## Step 6: Test the Setup

1. Commit and push the workflow file to your `main` branch:

   ```bash
   git add .github/workflows/deploy.yml
   git commit -m "Add CI/CD deployment workflow"
   git push origin main
   ```

2. Go to the **Actions** tab in your GitHub repository to confirm the workflow runs successfully.

3. Access your CloudFront domain (e.g., `https://d123abc.cloudfront.net`) to verify your site is live.

---

## Notes

- The workflow automatically deploys on every push to `main`.
- If you need to invalidate the CloudFront cache after deployment, add an invalidation step to the workflow.
- Ensure your website files are in the root of your repository for the current sync path to work correctly.

## Lessons Learned

- 🔒 Use least-privilege IAM policies instead of full-access policies.
- 🔑 Prefer OAC (Origin Access Control) over bucket policies for CloudFront access.
- 🔄 Add CloudFront cache invalidation after each deployment to serve fresh content immediately.
- 📄 Set `index.html` as the Error document for SPAs to avoid 404s on sub-paths.
- ☁️ CloudFront distributions take 15+ minutes to deploy — plan ahead when making changes.
- 🧪 Test the S3 website endpoint directly before attaching CloudFront to isolate issues faster.
- 🎯 Set proper `Cache-Control` headers on static assets for optimal CDN caching.
- 👀 Use `aws s3 sync --dryrun` first to preview what will be uploaded.
