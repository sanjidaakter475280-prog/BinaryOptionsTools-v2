# CI / Build instructions for BinaryOptionsTools-v2

এই ফাইলটিতে লোকাল থেকে Github পর্যন্ত আপনি যে ধাপগুলো সম্পন্ন করেছেন এবং ভবিষ্যতে পুনরাবৃত্তির জন্য দরকারি নির্দেশনা দেওয়া আছে। এছাড়া GitHub Actions workflow ও কিছু troubleshooting টিপসও রয়েছে।

---

## সারসংক্ষেপ (Quick summary)
- লোকালি প্রজেক্ট প্রস্তুত → Git init → GitHub রিমোট যোগ → push
- GitHub Actions workflow যোগ করে Ubuntu (Python 3.10 + Rust) এ `maturin build --release` চালানো হয়
- Build হলে wheel (target/wheels/*.whl) artifact হিসেবে আপলোড হয় → GitHub Actions → Artifacts → Download

---

## 1) লোকাল Git ও GitHub সেটআপ (commands)

- রিপো ইনিট এবং প্রথম commit:
  ```
  git init
  git add .
  git commit -m "Add project"
  ```

- GitHub এ নতুন রেপো তৈরি করে রিমোট যোগ করুন (HTTPS উদাহরণ):
  ```
  git remote add origin https://github.com/<your-username>/<repo-name>.git
  git branch -M main
  git push -u origin main
  ```

- যদি permission/credential error পান (401/403) — Windows Credential Manager-এ পুরানো `github.com` এন্ট্রি ডিলিট করুন এবং PAT (Personal Access Token) ব্যবহার করুন:
  - GitHub → Settings → Developer settings → Personal access tokens → Generate new token (select `repo` scope)
  - push করার সময় Username = GitHub username, Password = PAT

- বিকল্প (স্তব্ধ): SSH কী ব্যবহার করলে একবার সেট করলে আর পাসওয়ার্ড লাগবে না:
  ```
  ssh-keygen -t ed25519 -C "you@example.com"
  ssh-add ~/.ssh/id_ed25519
  # সাভ করা public key GitHub → Settings → SSH and GPG keys এ যোগ করুন
  git remote set-url origin git@github.com:<your-username>/<repo-name>.git
  git push -u origin main
  ```

---

## 2) GitHub Actions workflow (add this file)
রিপো-র রুটে `.github/workflows/build-wheel.yml` হিসেবে যোগ করুন। (নীচে যে YAML আছে সেটাই ব্যবহার করুন — PROJECT_DIR যদি আলাদা হয় সেটা পরিবর্তন করে নিন।)

```yaml
name: Build Python wheels

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]
  workflow_dispatch:

env:
  PROJECT_DIR: BinaryOptionsToolsV2  # <-- change if your project folder name differs

jobs:
  build-wheel:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Cache pip
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/pyproject.toml') }}
          restore-keys: |
            ${{ runner.os }}-pip-

      - name: Cache cargo registry & git
        uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
          key: ${{ runner.os }}-cargo-reg-${{ hashFiles('**/Cargo.lock') }}
          restore-keys: |
            ${{ runner.os }}-cargo-reg-

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Set up Rust toolchain
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
          profile: minimal
          override: true

      - name: Install system build deps (apt)
        run: |
          sudo apt-get update
          sudo apt-get install -y --no-install-recommends build-essential pkg-config libssl-dev python3-dev python3-pip curl ca-certificates git
          sudo rm -rf /var/lib/apt/lists/*

      - name: Ensure pip up-to-date and install maturin
        run: |
          python3 -m pip install --upgrade pip
          python3 -m pip install "maturin>=1.7,<2.0"

      - name: Print versions (debug)
        run: |
          python --version
          rustc --version
          cargo --version

      - name: Build wheel(s) with maturin
        run: |
          cd "${{ env.PROJECT_DIR }}"
          maturin build --release

      - name: List produced wheels
        run: ls -la "${{ env.PROJECT_DIR }}/target/wheels" || true

      - name: Upload wheels artifact
        uses: actions/upload-artifact@v4
        with:
          name: wheels
          path: ${{ env.PROJECT_DIR }}/target/wheels/*.whl
```

---

## 3) Workflow চালানো ও artifact ডাউনলোড
- workflow সফল হলে: GitHub → Repo → Actions → Run → নিচে `Artifacts` অংশে `wheels` দেখবেন
- Download করে unzip করুন — `.whl` ফাইল পাবেন
- local install/টেস্ট:
  ```
  python3.10 -m venv venv
  source venv/bin/activate
  pip install path/to/<the>.whl
  python -c "import BinaryOptionsToolsV2; print('ok', getattr(BinaryOptionsToolsV2,'__version__','no __version__'))"
  ```

---

## 4) wheel filename থেকে কী বোঝা যায়
উদাহরণ: `BinaryOptionsToolsV2-0.2.1-cp38-abi3-manylinux_2_34_x86_64.whl`
- `cp38` → CPython 3.8 টার্গেট (কিন্তু)
- `abi3` → PEP 384 compatible ABI — সাধারণত Python 3.8+ (3.9, 3.10...) এ কাজ করে
- `manylinux_2_34_x86_64` → Linux x86_64 প্ল্যাটফর্মের জন্য

> নোট: Windows/macOS হলে উপযুক্ত প্ল্যাটফর্মের wheel দরকার হবে (উদাহরণ: `win_amd64`)

---

## 5) Troubleshooting (সাধারণ ইস্যুগুলি)
- setup-python error: যদি log-এ বলা হয় `Version '3.1' not found` — workflow-এ `python-version` ভুল আছে; `3.10` বা `3.x` দিন।
- `target/wheels` not found: নিশ্চিত করুন `PROJECT_DIR` ঠিক আছে (pyproject.toml ও Cargo.toml ঐ ফোল্ডারে আছে)
- OpenSSL/native build error: apt-এ `libssl-dev` ইনস্টল করা আছে; যদি নির্দিষ্ট crate missing হয়, log কপি করে দেখান
- Credential errors (401/403): Windows Credential Manager থেকে পুরানো github.com এন্ট্রি মুছে PAT ব্যবহার করুন
- যদি build বড় বা timeouts হয়: self-hosted runner বা build optimization বিবেচনা করুন

---

## 6) Optional: publish to PyPI or Hugging Face
- আমি চাইলে workflow-এ publish ধাপ যোগ করতে পারি — তার জন্য GitHub Secrets → `PYPI_API_TOKEN` (PyPI) বা `HF_TOKEN` (Hugging Face) যোগ করতে হবে। নির্দেশ চাইলে যোগ করে দেবো।

---

## 7) Commands cheat-sheet
- Add workflow file + push:
  ```
  git add .github/workflows/build-wheel.yml
  git commit -m "CI: build wheels with maturin (Python 3.10) and cache"
  git push
  ```

- Download artifact via GitHub CLI (optional):
  ```
  gh run list --repo <owner>/<repo>
  gh run download <run-id> --name wheels --repo <owner>/<repo>
  ```

---

## যদি আপনি চান
- আমি এই `CI_INSTRUCTIONS.md` ফাইলটি আপনার রিপোতে commit & push করে দিতে পারি (আপনি যদি অনুমতি দেন)।  
- অথবা আমি workflow-এ PyPI/HF auto-publish ধাপও যোগ করে দিতে পারি — চাইলে বলে দিন, আমি সেটাপ করে দেব (আপনি Secrets যোগ করবেন)।
