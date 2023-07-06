1. `brew edit php@7.4`
2. Look for `disable! date: "2022-11-28", because: :versioned_formula`. Change 2022 to 2024 (or later).
3. `HOMEBREW_NO_INSTALL_FROM_API=1 brew install --build-from-source php@7.4`