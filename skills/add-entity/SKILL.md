# /add-entity [name]

Scaffold a MikroORM entity (with optional translation entity) in an existing module.

```yaml
disable-model-invocation: true
```

## Arguments

- `$ARGUMENTS` = entity name in kebab-case singular (e.g., `article`, `event`, `testimonial`)

## Instructions

1. **Parse input**: Extract `<name>` from `$ARGUMENTS`. Convert to kebab-case singular if not already. Derive:
   - `kebab` = kebab-case singular (e.g., `article`)
   - `PascalSingular` = PascalCase singular (e.g., `Article`)
   - `snake_plural` = snake_case plural for table name (e.g., `articles`)

2. **Ask the user**:
   - Which **existing module** to place the entity in (e.g., `contents`, `pets`, `product`)
   - Whether to include **translations** (yes/no)
   - Whether to add **project scoping** (`@ManyToOne(() => Project)`) — default yes

3. **Read reference files** to match the exact current patterns:
   - `src/common/entities/Base-entity.ts`
   - `src/pets/entities/pet.entity.ts`
   - `src/pets/entities/pet-translation.entity.ts` (if translations)

4. **Create files**:

### Main entity (`src/<module>/entities/<name>.entity.ts`)
- Extend `BaseEntity` from `@/common/entities/Base-entity`
- `@ObjectType('PascalName')` + `@Entity({ tableName: 'snake_plural' })`
- If project-scoped: `@Unique({ properties: ['project', 'slug'] })` + `@ManyToOne(() => Project)` with `@Index()`
- `slug` field with `@Index()`
- If translations: `@OneToMany({ entity: () => TranslationEntity, mappedBy: (x) => x.<parent>, cascade: [Cascade.ALL] })` initialized as `= new Collection<TranslationType>(this)`
- Add a few placeholder fields with TODO comments for the user to customize

### Translation entity (`src/<module>/entities/<name>-translation.entity.ts`) — if translations
- Extend `BaseEntity`
- `@ObjectType('PascalNameTranslation')` + `@Entity({ tableName: '<name>_translations' })`
- `@Unique({ properties: ['<parent>', 'language'] })`
- `@ManyToOne(() => ParentEntity)` with `@Index()` — the parent field name matches the entity name in camelCase
- `@ManyToOne(() => LanguageEntity)` with `@Index()` — import from `@/language/entities/language.entity`
- Standard translated fields: `title: string`, `description?: string` (text type), `metaTitle?: string`, `metaDescription?: string` (text type)

### Key patterns
- All decorators: both `@Field()` (GraphQL) and `@Property()` / `@ManyToOne()` etc. (MikroORM)
- Nullable fields: `@Field({ nullable: true })` + `@Property({ nullable: true })` + `?:` TypeScript
- Text fields: `@Property({ type: 'text', nullable: true })`
- Enums: define enum, `registerEnumType(MyEnum, { name: 'MyEnum' })`, use `@Enum({ items: () => MyEnum })`
- Collections always initialized: `= new Collection<T>(this)`
- Import `Collection`, `Cascade` from `@mikro-orm/core`

### DTO sanitize decorators

For every **Create** input DTO (and inherited by `PartialType` Update DTOs), decorate each
user-supplied text field. See `.claude/rules/sanitize.md` for the full heuristic.

```ts
import { SanitizeRichText, SanitizePlainText } from '@/common/decorators/sanitize.decorator'

@InputType('CreateEntityTranslationInput')
export class CreateEntityTranslationInput {
  @Field()
  @IsString()
  @SanitizePlainText()   // title — no HTML
  title!: string

  @Field({ nullable: true })
  @IsString()
  @IsOptional()
  @SanitizeRichText()    // description — HTML body
  description?: string

  @Field({ nullable: true })
  @IsString()
  @IsOptional()
  @SanitizePlainText()   // metaTitle, metaDescription — never HTML
  metaTitle?: string
}
```

Quick rule: `description`/`htmlContent`/`body` → `@SanitizeRichText()`. Everything else
(titles, names, meta, notes, addresses) → `@SanitizePlainText()`. Identifiers (slug, email,
URL) → no sanitize decorator.

### Cache pattern registration

If this entity will have a public read endpoint, register its cache invalidation pattern in
`src/revalidation/services/revalidation.service.ts` (`ENTITY_CACHE_PATTERNS`):

```ts
[EntityType.MY_ENTITY]: (projectId) => [`public:my-entities:${projectId}:*`]
```

Add `public:sitemap:${projectId}:*` to the same list if the entity surfaces in the sitemap.
See `.claude/rules/caching.md` for details.

## Next steps (print to user)

After creating the entity:

1. **Add entity to module** — add to `MikroOrmModule.forFeature([...])` in `src/<module>/<module>.module.ts`
2. **Create migration** — `pnpm migration:create` then review the generated SQL
3. **Run migration** — `pnpm migration:up`
4. **Customize fields** — replace placeholder fields with domain-specific properties
5. **Add relations** — if this entity relates to others, add `@ManyToOne` / `@OneToMany` as needed
6. **Sanitize DTOs** — apply `@SanitizeRichText()` / `@SanitizePlainText()` on Create input fields per `.claude/rules/sanitize.md`
7. **Register cache pattern** — if the entity has a public read endpoint, add to `ENTITY_CACHE_PATTERNS` per `.claude/rules/caching.md`
