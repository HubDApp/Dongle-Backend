# Contributing to Dongle Backend

Thank you for your interest in contributing to Dongle! We welcome contributions from developers, designers, and community members.

## Getting Started

1. **Fork the Repository**
   ```bash
   git clone https://github.com/yourusername/dongle-backend.git
   cd dongle-backend
   ```

2. **Install Dependencies**
   ```bash
   pnpm install
   ```

3. **Set Up Development Environment**
   ```bash
   cp .env.example .env.local
   # Edit .env.local with your local database URL and contract IDs
   pnpm run db:migrate
   ```

4. **Create a Feature Branch**
   ```bash
   git checkout -b feature/your-feature-name
   ```

## Development Workflow

### Code Style & Linting

The project uses ESLint and Prettier for code formatting.

```bash
# Check code style
pnpm run lint

# Auto-fix style issues
pnpm run lint:fix

# Type checking
pnpm run typecheck
```

### Commit Guidelines

Use Conventional Commits format:

```
type(scope): subject

body

footer
```

**Types**:
- `feat`: A new feature
- `fix`: A bug fix
- `docs`: Documentation only changes
- `style`: Changes that don't affect code meaning (whitespace, formatting)
- `refactor`: Code changes that neither fix bugs nor add features
- `perf`: Code changes that improve performance
- `test`: Adding or updating tests
- `chore`: Changes to build process or dependencies

**Examples**:
```
feat(projects): add full-text search to project listings
fix(reviews): fix rating aggregation calculation
docs(api): update project endpoint documentation
test(services): add tests for verification workflow
```

### Testing

Write tests for new features and bug fixes.

```bash
# Run all tests
pnpm run test

# Run tests in watch mode
pnpm run test:watch

# Run specific test file
pnpm run test -- services/project.service.test.ts

# Generate coverage report
pnpm run test:coverage
```

**Test Guidelines**:
- Aim for >80% code coverage
- Write unit tests for services and utilities
- Write integration tests for API endpoints
- Mock external dependencies (Soroban, databases)
- Use descriptive test names

**Example Test**:
```typescript
describe('ProjectService', () => {
  describe('search', () => {
    it('should return projects matching search query', async () => {
      const results = await projectService.search('stellar');
      expect(results).toHaveLength(2);
      expect(results[0].name).toContain('stellar');
    });

    it('should return empty array when no matches', async () => {
      const results = await projectService.search('nonexistent');
      expect(results).toHaveLength(0);
    });
  });
});
```

## Creating a Pull Request

1. **Push Your Branch**
   ```bash
   git push origin feature/your-feature-name
   ```

2. **Create a Pull Request**
   - Title: Use conventional commit format
   - Description: Explain what changes and why
   - Reference related issues: "Fixes #123"

3. **PR Description Template**
   ```markdown
   ## Description
   Brief explanation of the changes

   ## Type of Change
   - [ ] Bug fix
   - [ ] New feature
   - [ ] Breaking change
   - [ ] Documentation update

   ## Related Issues
   Fixes #123

   ## How to Test
   Steps to verify the changes work correctly

   ## Checklist
   - [ ] Code follows linting rules (`pnpm run lint`)
   - [ ] Types check (`pnpm run typecheck`)
   - [ ] Tests added/updated and passing (`pnpm run test`)
   - [ ] Documentation updated
   - [ ] No console warnings or errors
   ```

4. **Code Review**
   - Maintainers will review your PR
   - Address feedback and push updates
   - All checks must pass before merging

## Adding New Features

### Adding a New Endpoint

1. **Design the endpoint** — Document in [docs/API.md](API.md)
   ```markdown
   ## Create New Feature
   
   **Endpoint**: `POST /api/features`
   **Authentication**: Required (Bearer token)
   
   ### Request Body
   ```

2. **Create the route** — `src/api/features/routes.ts`
   ```typescript
   export const router = express.Router();
   
   router.post('/', async (req, res, next) => {
     try {
       const result = await featureController.create(req.body);
       res.status(201).json(result);
     } catch (err) {
       next(err);
     }
   });
   ```

3. **Create the controller** — `src/api/features/controller.ts`
   ```typescript
   export class FeatureController {
     async create(data: CreateFeatureDTO) {
       validateCreateFeatureInput(data);
       return await featureService.create(data);
     }
   }
   ```

4. **Create the service** — `src/services/feature.service.ts`
   ```typescript
   export class FeatureService {
     async create(data: CreateFeatureDTO): Promise<Feature> {
       // Business logic here
       return await featureRepository.create(data);
     }
   }
   ```

5. **Create the repository** — `src/repositories/feature.repository.ts`
   ```typescript
   export class FeatureRepository {
     async create(data: CreateFeatureDTO): Promise<Feature> {
       const result = await db.query(
         'INSERT INTO features (...) VALUES (...) RETURNING *',
         [...]
       );
       return result.rows[0];
     }
   }
   ```

6. **Add tests** — `tests/integration/features.test.ts`
   ```typescript
   describe('POST /api/features', () => {
     it('should create a new feature', async () => {
       const response = await request(app)
         .post('/api/features')
         .send({ name: 'Test Feature' })
         .expect(201);
       
       expect(response.body.data.name).toBe('Test Feature');
     });
   });
   ```

7. **Update documentation** — Add to [docs/API.md](API.md)

### Adding a Database Migration

```bash
# Create migration file
pnpm run db:create-migration CreateFeaturesTable
```

Edit `src/database/migrations/YYYYMMDDHHMMSS_CreateFeaturesTable.ts`:

```typescript
export async function up(db: Database): Promise<void> {
  await db.query(`
    CREATE TABLE features (
      id SERIAL PRIMARY KEY,
      name VARCHAR(255) NOT NULL,
      created_at TIMESTAMP DEFAULT NOW()
    )
  `);
}

export async function down(db: Database): Promise<void> {
  await db.query('DROP TABLE features');
}
```

## Project Structure Guidelines

Follow these conventions:

- **Services** (`src/services/*.ts`): Pure business logic, no side effects
- **Repositories** (`src/repositories/*.ts`): Data access layer only
- **Controllers** (`src/api/*/controller.ts`): Orchestrate services, format responses
- **Routes** (`src/api/*/routes.ts`): Define HTTP interface
- **Types** (`src/models/*.ts`): Domain types and interfaces
- **Tests** (`tests/`): Mirror source structure with `.test.ts` suffix

## Documentation

### Writing Documentation

- Use Markdown for documentation files
- Include code examples where helpful
- Link to related documents
- Update [README.md](../README.md) for API/feature changes
- Keep documentation in sync with code

### Example Documentation Entry

```markdown
## Feature Name

Brief description of what this does.

### Usage

How to use this feature with code examples:

\`\`\`typescript
const result = await featureService.doSomething();
\`\`\`

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| ... | ... | ... | ... |

### Returns

What the function returns.

### Examples

Real-world usage examples.
```

## Security Considerations

When contributing code, please:

1. **Never commit secrets** — Use environment variables for sensitive data
2. **Validate input** — All user inputs should be validated/sanitized
3. **Use parameterized queries** — Prevent SQL injection
4. **Check authentication** — Verify permissions before sensitive operations
5. **Log securely** — Don't log sensitive data (passwords, keys, tokens)
6. **Review dependencies** — Check `pnpm audit` before adding new packages

## Performance Guidelines

- Avoid N+1 queries — batch database queries
- Use indexes on frequently queried columns
- Implement caching for expensive operations
- Paginate large result sets
- Profile slow operations with `npm run perf`
- Keep response payloads small — exclude unnecessary fields

## Reporting Issues

### Bug Reports

Include:
1. Reproduction steps
2. Expected behavior
3. Actual behavior
4. Environment (Node version, OS, database)
5. Error logs/stack trace

### Feature Requests

Include:
1. Problem you're solving
2. Proposed solution
3. Alternative approaches considered
4. Related issues/PRs

## Getting Help

- **Discussions**: Use GitHub Discussions for questions
- **Issues**: Open an issue for bugs
- **Chat**: Join our community chat (if available)
- **Docs**: Check [docs/](.) and [README.md](../README.md) first

## Code of Conduct

- Be respectful and inclusive
- Assume good intent
- Provide constructive feedback
- Report violations to maintainers

## Recognition

Contributors are recognized in:
- GitHub Contributors page
- Release notes (for significant contributions)
- CONTRIBUTORS.md file (coming soon)

## License

By contributing, you agree that your contributions will be licensed under the same license as the project (MIT).

---

**Thank you for contributing to Dongle! 🙌**
