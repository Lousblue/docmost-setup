# docmost-setup
Deploy Docmost with Docker Compose, backups included. Postgres, Redis and the app; secrets asked when it runs or taken from the environment, SMTP optional. --backup dumps the database and the uploads with rotation, --cron schedules it daily, --restore puts a backup back, --update backs up first, then pulls.
