# 📦 Network Device Configuration Backup (Jenkins + Ansible + Git)

이 프로젝트는 **네트워크 장비 구성 백업**을 자동화하기 위한 파이프라인입니다.  
Jenkins에서 주기적으로 실행되어, 구성 백업을 수행하고 GitHub 저장소에 push합니다.

## 🔧 구성 요소

- **Jenkins**: 주기적인 실행 자동화 (예: 매일 22시)
- **Ansible**: 멀티 벤더 장비 구성 백업 플레이북 실행
- **Jinja2 (j2cli)**: 동적 인벤토리 파일 생성
- **Git**: 백업 결과 버전 관리 및 GitHub에 push

## 🧩 백업 스크립트 개요

```bash
#!/bin/bash
set -e

export PATH=/usr/local/bin:$PATH
export HOME=/var/lib/jenkins
export SSH_AUTH_SOCK=/tmp/ssh-agent.sock

# SSH 인증서 준비
[ -S $SSH_AUTH_SOCK ] && rm -f $SSH_AUTH_SOCK
eval "$(ssh-agent -a $SSH_AUTH_SOCK)"
ssh-add $HOME/.ssh/id_rsa

# j2 템플릿 → 인벤토리 생성
INVENTORY_FILE="$WORKSPACE/nw-temp-inventory.ini"
j2 --format=env /etc/ansible/inventory/nw-inventory.j2 > "$INVENTORY_FILE"

# 백업 디렉토리 생성
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="/etc/ansible/results/nw-backup/${TIMESTAMP}"
mkdir -p "$BACKUP_DIR"

# 구성 백업 실행
ansible-playbook -i "$INVENTORY_FILE" \
  /etc/ansible/roles/config-backup/playbook.yml \
  -e "backup_dir=${BACKUP_DIR}"

# 정리 및 결과 업로드
rm -f "$INVENTORY_FILE"
cd /etc/ansible/results
git config --global --add safe.directory /etc/ansible/results/nw-backup
git add .
git commit -m "자동 백업: ${TIMESTAMP}" || echo "변경 사항 없음"
git push origin main
```

## 📁 백업 결과 저장 구조

모든 백업 파일은 타임스탬프 기반 디렉토리에 저장됨
```bash
/etc/ansible/results/nw-backup/
└── 20250726-213416/
    ├── c8000v-dc1_20250726-213416.cfg
    ├── c8000v-dc2_20250726-213416.cfg
    ├── tl-fw-dc1_20250726-213416.cfg
    └── tl-fw-dc2_20250726-213416.cfg
```

## ⏰ 실행 주기 예시 (Jenkins cron)
매 1시간마다 config 백업 적용!

| 주기            | 표현         | 의미                     |
|-----------------|--------------|--------------------------|
| 매 1시간        | `H * * * *`  | 매 시간 랜덤 분마다 실행 |
| 매일 22시       | `0 22 * * *` | 매일 밤 10시 실행        |
| 매주 일요일 3시 | `0 3 * * 0`  | 매주 일요일 새벽 3시 실행|
| 매월 1일 자정   | `0 0 1 * *`  | 매월 1일 0시 실행         |
