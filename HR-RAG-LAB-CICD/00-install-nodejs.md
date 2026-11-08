# 00 Install Node.js

The setup builds the chatbot's web page with a tool called `npm`, which installs JavaScript packages. `npm` comes with Node.js, so you need Node.js version 20, or 22 and later. The reason is the web page's build tool, Vite 6: it supports Node.js 18, 20 and 22 or later, and not the odd versions 19 and 21. Node.js 18 is no longer maintained, so start at 20. The current LTS (long-term support) release works. The setup in document 02 checks the version for you.

## Windows (Chocolatey)

Chocolatey is a package manager for Windows: it installs programs with one command. Open PowerShell as an administrator: right-click **PowerShell** and choose **Run as administrator**. Then run:

```powershell
choco install nodejs-lts -y
```

If you do not have Chocolatey, download the LTS installer from [nodejs.org](https://nodejs.org/en/download) and run it.

## Mac (Homebrew)

```bash
brew install node
```

If you do not have Homebrew, download the LTS installer from [nodejs.org](https://nodejs.org/en/download) and run it.

## Check

Open a new terminal, then run:

```bash
node --version
npm --version
```

**Expected:** `node --version` prints `v20`, `v22` or higher, such as `v24.22.0`. `npm --version` prints a version number, such as `11.19.1`.

**Next:** [01 Import and clone](01-import-and-clone.md)
