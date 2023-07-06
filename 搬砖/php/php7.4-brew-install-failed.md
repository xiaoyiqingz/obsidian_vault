1. `brew edit php@7.4`
2. Look for `disable! date: "2022-11-28", because: :versioned_formula`. Change 2022 to 2024 (or later).
3. `HOMEBREW_NO_INSTALL_FROM_API=1 brew install --build-from-source php@7.4`


## brew php 降级icu4c

1. cd $(brew --prefix)/Homebrew/Library/Taps/homebrew/homebrew-core/Formula
2.  git log --follow icu4c.rb 
3.  找到70版本icu4c，checkout commit id 
4.  HOMEBREW_NO_AUTO_UPDATE=1   brew reinstall icu4c