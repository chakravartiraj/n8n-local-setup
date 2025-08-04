# Contributing to N8N Local Setup

Thank you for your interest in contributing to this N8N local setup project!

## How to Contribute

### 🐛 Reporting Bugs
- Use the GitHub Issues tab
- Include your Docker and Docker Compose versions
- Provide logs if applicable
- Describe the expected vs actual behavior

### 💡 Suggesting Enhancements
- Check existing issues first
- Clearly describe the enhancement
- Explain why it would be useful

### 🔧 Pull Requests
1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Make your changes
4. Test the Docker Compose setup
5. Update documentation if needed
6. Commit your changes (`git commit -m 'Add amazing feature'`)
7. Push to the branch (`git push origin feature/amazing-feature`)
8. Open a Pull Request

## Development Guidelines

### Testing Changes
```bash
# Test the configuration
docker compose config

# Test the deployment
docker compose up -d

# Check health
docker compose ps
```

### Documentation
- Update README.md for configuration changes
- Add comments to docker-compose.yml for complex configurations
- Include examples where helpful

## Code of Conduct
- Be respectful and inclusive
- Focus on constructive feedback
- Help others learn and grow

Thank you for contributing! 🚀
