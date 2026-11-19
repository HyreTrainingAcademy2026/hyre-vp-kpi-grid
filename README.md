name: Sync KPIs from ClickUp & deploy

on:
  schedule:
    - cron: "*/30 * * * *"   # every 30 minutes
  workflow_dispatch: {}
  repository_dispatch:
    types: [clickup-doc-updated]
  push:
    branches: [main]

permissions:
  contents: write
  pages: write
  id-token: write

concurrency:
  group: kpi-sync
  cancel-in-progress: false

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - name: Pull latest Role Metrics Library from ClickUp
        env:
          CLICKUP_API_TOKEN: ${{ secrets.CLICKUP_API_TOKEN }}
        run: node scripts/sync.mjs
      - name: Commit changes
        run: |
          if [ -n "$(git status --porcelain data/)" ]; then
            git config user.name "kpi-sync-bot"
            git config user.email "kpi-sync-bot@users.noreply.github.com"
            git add data/kpis.json
            git commit -m "Sync KPIs from ClickUp"
            git push
          else
            echo "No KPI changes."
          fi
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: .
      - id: deployment
        uses: actions/deploy-pages@v4
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
