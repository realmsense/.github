# realmsense
good luck, half this code is absolute dog 💩

This project *should* have been a monorepo, instead I used a script to simulate one:
<details>
  <summary>batch.sh</summary>
  
  ```sh
  GITEA_USER="Extacy"
  GITEA_PASS=""
  GITEA_URL=""
  GITEA_ORG="realmsense"

  if [ -z $1 ]; then
      echo "Usage: $0 [clone | pull | push | env <file>]"
      exit 1
  fi

  pull_all() {

      get_repos
      for repo_name in "${repos[@]}"; do
          file_exists $repo_name
          cd $repo_name

          git pull
          get_submodules
          
          for submodule in "${submodules[@]}"; do
              file_exists $submodule
              local cwd=$(pwd)
              cd $submodule
              git pull
              cd $cwd
          done

          cd ..
      done
  }

  push_all() {

      get_repos
      for repo_name in "${repos[@]}"; do
          file_exists $repo_name
          cd $repo_name

          get_submodules
          for submodule in "${submodules[@]}"; do
              file_exists $submodule
              local cwd=$(pwd)
              cd $submodule
              git push
              cd $cwd
          done
          git push
          cd ..
      done
  }

  link_env() {

      local env_file=$2
      file_exists $env_file

      get_repos
      for repo_name in "${repos[@]}"; do
          file_exists $repo_name

          local link="${repo_name}\\.env"
          local target=$(win_dir $env_file)
          mklink "//H" $link $target
      done
  }

  clone_all() {

      echo -e "\n[~] Cloning all repos..."

      get_repos

      # Clone all
      for repo_name in "${repos[@]}"; do
          local clone_url="https://${GITEA_USER}:${GITEA_PASS}@${GITEA_URL}/${GITEA_ORG}/${repo_name}"
          echo -e "\n[~] Cloning repo: $repo_name"
          git clone -c core.symlinks=true $clone_url
      done

      # Link Submodules
      for repo_name in "${repos[@]}"; do
          cd $repo_name
          get_submodules

          for submodule in "${submodules[@]}"; do
              echo -e "\n[~] (${repo_name}) Linking submodule: ${submodule}"
              rm -r $submodule

              local link=$(win_dir $submodule)
              local target="$(win_dir ../$(basename $submodule))"
              mklink "//J" $link $target
          done
          cd ..
      done

      # Install NPM Packages
      for repo_name in "${repos[@]}"; do
          echo -e "\n[~] (${repo_name}) Installing npm dependencies"
          cd $repo_name
          if [ -f ./package.json ]; then
              npm install
          fi
          cd ..
      done

      echo -e "\n[~] Done"
  }

  win_dir() {
      # Replace forward slashes to a backslash
      echo $(sed 's:/:\\:g' <<< $1)
  }

  # Delete an element from an array by value and shift indexes
  arr_del_elem() {
      local -n array=$1
      local delete=$2

      for i in "${!array[@]}"; do
          if [[ ${array[i]} = $delete ]]; then
              unset "array[i]"
          fi
      done
  }

  file_exists() {
      local file=$1
      if [ ! -e "$file" ]; then
          echo "$file does not exist! Exiting."
          exit 1
      fi 
  }

  get_repos() {
      # API Request to get organization repositories
      local repos_json=$(curl --silent -X "GET" \
      "https://${GITEA_USER}:${GITEA_PASS}@${GITEA_URL}/api/v1/orgs/${GITEA_ORG}/repos" \
      -H "accept: application/json")

      repos=($(jq -r -c ".[].name" <<< $repos_json | tr -d '\r'))
      arr_del_elem repos realmsense # Remove the "realmsense" (this) repo from the list
  }

  get_submodules() {
      submodules=($(git config --file .gitmodules --get-regexp path | awk '{ print $2 }'))
  }

  mklink() {
      local LINK_BAT="link.bat"
      {
          # echo "@echo off"
          echo "mklink %*"
      } > $LINK_BAT

      ./$LINK_BAT $@
      rm $LINK_BAT
  }

  if [ $1 == "clone" ]; then
      clone_all "$@"
  elif [ $1 == "pull" ]; then
      pull_all "$@"
  elif [ $1 == "push" ]; then
      push_all "$@"
  elif [ $1 == "env" ]; then
      link_env "$@"
  else
      echo "Unknown command: $1"
  fi
  ```
</details>

<details>
  <summary>.env</summary>
  
  ```sh
  PRODUCTION=                 # boolean

  # URL
  HTTP=                       # "https://" or "http://"
  API_URL=                    # "api.realmsense.cc"
  WEBSITE_URL=                # "realmsense.cc"
  EXTRACTOR_URL=              # "rotmg.extacy.cc"
  PMA_URL=                    # "pma.realmsense.cc"
  TRAEFIK_URL=                # "traefik.realmsense.cc"

  # Extractor
  EXTRACTOR_UPDATE_PATH=      # "/production/client/current"
  EXTRACTOR_WEBHOOK_URL=      # Discord webhook URL
  EXTRACTOR_WEBHOOK_MESSAGE=  # "<@&848528094918737931>"

  EXTRACTOR_IDA_ENABLED=      # boolean
  EXTRACTOR_IDA_AUTH=         # auth string
  EXTRACTOR_IDA_SERVER=       # "http://ida:5000/ida/command"
  EXTRACTOR_IDA_CMD=          # "ida.sh"
  EXTRACTOR_IDA_WORKDIR=      # "/usr/src/ida"

  # Traefik
  CF_DNS_API_TOKEN=           # API token with Zone.DNS:Edit permission
  CF_ZONE_API_TOKEN=          # API token with Zone.Zone:Read permission
  ACME_EMAIL=                 # Email address to receive notifications from Lets Encrypt
  TRAEFIK_DASHBOARD_AUTH=     # htpasswd -nB username

  # JWT
  JWT_SECRET=                 # Random password string
  JWT_EXPIRATION=             # in seconds

  # Authkey
  TrackerPlugin=              # Key for protected routes
  TrackerDiscord=             # Key for protected routes
  RaidTracker=                # Key for protected routes

  # Discord
  DISCORD_INVITE=             # Discord invite link
  DISCORD_CLIENTID=           # OAuth2 Client ID
  DISCORD_CLIENTSECRET=       # OAuth2 Client Secret
  DISCORD_REDIRECTURI=        # OAuth2 Redirect URL
  DISCORD_BOT_TOKEN=          # Discord Bot Token

  # Discord Raid Tracker
  RAID_TRACKER_CHANNEL=       # Announcement channel for Discord runs
  RAID_TRACKER_SELFBOT_TOKEN= # Self Bot token used to track Discord runs

  # Nrelay Tracker
  TRACKER_API_ENABLED=        # boolean
  TRACKER_REALMS_ENABLED=     # boolean
  TRACKER_ADMIN_NAME=         # username of player that can send commands to the bots in game

  # Database
  DB_DEFAULT=                 # "rs_default"
  DB_TRACKER=                 # "rs_tracker"
  DB_CUSTOMERS=               # "rs_customers"

  DB_HOST=                    # "database"
  DB_PORT=                    # "3306"
  DB_USERNAME=
  DB_PASSWORD=
  DB_ROOT_PASSWORD=

  DB_MIGRATIONS=              # boolean - should DB Migrations be auto run on start
  DB_SYNCHRONIZE=             # boolean - force synchronize database with local entities. DO NOT enable in Production!!
  DB_LOGGING=                 # boolean - enable logging of SQL commands

  ```
</details>
