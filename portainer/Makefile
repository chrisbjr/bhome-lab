up:
	@docker compose up -d

up-cf:
	@docker compose --profile cloudflare up -d

up-recreate:
	@docker compose up --force-recreate --remove-orphans -d

down:
	@docker compose down

destroy:
	@docker compose down --rmi all --volumes --remove-orphans
