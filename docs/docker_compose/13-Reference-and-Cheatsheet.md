# Docker Compose Reference & Cheatsheet

## Command Reference
```bash
docker compose up
docker compose down
docker compose ps
docker compose logs
docker compose exec <service> <command>
docker compose build
docker compose pull
docker compose push
docker compose stop
docker compose restart
```

## YAML Reference
- `services`, `volumes`, `networks`, `secrets`, `configs`, `depends_on`, `healthcheck`, `deploy`, `build`, `environment`, `command`, `entrypoint`, `profiles`

## Quick Tips
- Use `.env` files for configuration
- Use profiles for optional services
- Use multiple Compose files for overrides

## Troubleshooting
- Check logs and container status
- Use `docker inspect` for details

## Resources
- [Official Docs](https://docs.docker.com/compose/)
- [Compose Specification](https://compose-spec.io/)

---
*End of Docker Compose Learning Path*
