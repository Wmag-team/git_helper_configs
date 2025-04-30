#!/bin/bash

# Включить выход при ошибках и вывод выполняемых команд (для отладки)
set -e
# set -x  # раскомментировать для отладки

# Конфигурация
REPO_DIR="/home/StroyMonitoring2.0"
DOCKER_DIR="$REPO_DIR/docker"

# Цвета для вывода
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Функция для вывода ошибок
error() { echo -e "${RED}!! $1${NC}"; exit 1; }
info() { echo -e "${GREEN}>> $1${NC}"; }
warning() { echo -e "${YELLOW}>> $1${NC}"; }

# Главный заголовок
echo -e "\n=== ${GREEN}BRANCH DEPLOYMENT SCRIPT${NC} ==="
echo "==============================="

# Проверка и переход в директорию репозитория
info "Changing to repository directory: $REPO_DIR"
cd "$REPO_DIR" || error "CRITICAL ERROR: Failed to enter $REPO_DIR"

# Проверка, что это git репозиторий
[ -d .git ] || git rev-parse --git-dir > /dev/null 2>&1 || error "$REPO_DIR is not a git repository!"

# 1. Остановка контейнеров
info "Stopping docker containers..."
cd "$DOCKER_DIR" || error "Docker directory not found!"
docker-compose down || warning "Some containers could not be stopped"

cd "$REPO_DIR" || error "CRITICAL ERROR: Failed to enter $REPO_DIR"

# Функция выбора ветки только из origin
select_remote_branch() {
    info "Fetching branch list from remote repository..."
    git fetch origin > /dev/null 2>&1 || error "Failed to fetch from origin"
    
    # Получаем только ветки из origin, исключая HEAD
    branches=$(git ls-remote --heads origin | awk -F'refs/heads/' '{print $2}' | sort)
    
    [ -z "$branches" ] && error "Failed to get branch list from origin!"

    echo -e "\n${GREEN}Available remote branches:${NC}"
    PS3=">> Select branch number to deploy: "
    select branch in $branches; do
        if [ -n "$branch" ]; then
            info "Selected branch: $branch"
            BRANCH=$branch
            break
        else
            warning "Invalid selection. Please try again."
        fi
    done
}

# Определение ветки для деплоя
if [ -z "$1" ]; then
    select_remote_branch
else
    BRANCH=$1
    info "Verifying branch $BRANCH exists in origin..."
    git fetch origin > /dev/null 2>&1
    if ! git ls-remote --exit-code --heads origin "$BRANCH" > /dev/null 2>&1; then
        warning "Branch $BRANCH not found in remote repository."
        select_remote_branch
    fi
fi

cd "$REPO_DIR" || error "CRITICAL ERROR: Failed to enter $REPO_DIR"

# 2. Синхронизация с выбранной веткой
info "Syncing with origin/$BRANCH..."
git fetch origin "$BRANCH" --force
git reset --hard "origin/$BRANCH" || error "Failed to reset to branch $BRANCH"

# 3. Установка прав
info "Setting file permissions..."
sudo chmod -R 775 laravel/storage laravel/bootstrap/cache
sudo find laravel/storage -type f -exec chmod 664 {} \;
sudo find laravel/bootstrap/cache -type d -exec chmod 775 {} \;
sudo rm -f laravel/storage/logs/*.log

# 4. Перезапуск контейнеров
info "Rebuilding and restarting containers..."
cd "$DOCKER_DIR" || error "Docker directory not found!"
docker-compose build --no-cache && docker-compose up -d || error "Failed to start containers"

# Очистка Docker
info "Cleaning up Docker system..."
docker system prune -a --force || warning "Docker prune failed"

# Завершение
echo -e "\n${GREEN}>> DEPLOYMENT COMPLETED SUCCESSFULLY!${NC}"
info "Current branch: $BRANCH"
info "All local changes were discarded!"
