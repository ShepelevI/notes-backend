**Система контроля версий** — программа, которая хранит разные версии одного документа, позволяет переключаться между ними, вносить и отслеживать изменения.

#### Полезные команды:

-     find /путь/к/директориям -type d -name ".git"
-     rm -rf .git
-     rm -rf .git*
-     ls -lah
-     git config --global user.name "имя"
-     git config --global user.email "почта"
-     git config --global init.defaultBranch main
-     git config --global core.autocrlf input
-     git config --global core.safecrlf warn
-     ls -al ~/.ssh
-     ssh-keygen -t ed25519 -C "почта"

  если система не поддерживает, то:

-     -ssh-keygen -t rsa -b 4096 -C "почта"
-     eval "$(ssh-agent -s)"
-     ssh-add ~/.ssh/id_ed25519
-     clip < ~/.ssh/id_ed25519.pub

  если не сработало, то:

-     ~ cat ~/.ssh/id_ed25519.pub


