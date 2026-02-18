# Create the GitHub repository and push this project

Follow these steps to publish this folder as a **public** GitHub repo.

## 1. Create the repository on GitHub

1. Go to **https://github.com/new**
2. **Repository name:** e.g. `dataset-football-ligue1` or `ligue1-dataviz-2025`
3. **Description (optional):** e.g. `Ligue 1 Power BI DataViz project — season 2023–2024`
4. Choose **Public**
5. **Do not** add a README, .gitignore, or license (this folder already has them).
6. Click **Create repository**.

## 2. Push from your computer

In PowerShell, from this project folder, run:

```powershell
cd "c:\Users\narek.pirumyan\Desktop\IAE\2025\DataViz\Project\dataset_football_ligue1"

# Add GitHub as remote (replace YOUR_USERNAME and REPO_NAME with your repo)
git remote add origin https://github.com/YOUR_USERNAME/REPO_NAME.git

# Push (first time: set upstream branch)
git branch -M main
git push -u origin main
```

Replace:
- `YOUR_USERNAME` with your GitHub username  
- `REPO_NAME` with the repository name you chose (e.g. `dataset-football-ligue1`)

If GitHub asks for credentials, use a **Personal Access Token** (not your password):  
**GitHub → Settings → Developer settings → Personal access tokens**.

## 3. Using SSH instead of HTTPS

If you use SSH keys:

```powershell
git remote add origin git@github.com:YOUR_USERNAME/REPO_NAME.git
git branch -M main
git push -u origin main
```

---

After pushing, your repo will be available at:  
`https://github.com/YOUR_USERNAME/REPO_NAME`
