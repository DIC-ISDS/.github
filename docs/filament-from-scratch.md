# Building a Filament Panel from Scratch

Filament is a server-driven UI framework for Laravel. You describe screens in PHP
— forms, tables, actions — and it renders them with Livewire, Alpine and Tailwind.
No Blade to write, no JavaScript to ship.

This walks through a panel from an empty Laravel app to a working CRUD screen, in
the order you'd actually do it. Everything here follows the
[official Filament docs](https://filamentphp.com/docs); the commands and code
match **Filament v5** (5.7 at the time of writing).

A warning before you start copying from elsewhere: v3 and v4 tutorials will not
line up with this. Namespaces moved, resource directories got nested, and table
actions were renamed. [Section 14](#14-coming-from-v3-or-v4) lists what changed.

If this project runs in the Docker setup, see
[docker-environment.md](docker-environment.md) — every `php artisan` command
below has to run inside the PHP container.

---

## Contents

- [1. What you need first](#1-what-you-need-first)
- [2. Install Filament](#2-install-filament)
  - [Running the commands in Docker](#running-the-commands-in-docker)
- [3. What the installer created](#3-what-the-installer-created)
- [4. Your first resource](#4-your-first-resource)
  - [The model and migration](#the-model-and-migration)
  - [Generating the resource](#generating-the-resource)
  - [What each generated file does](#what-each-generated-file-does)
- [5. The form](#5-the-form)
  - [Layout components](#layout-components)
  - [Showing a field on only one page](#showing-a-field-on-only-one-page)
- [6. The table](#6-the-table)
- [7. Relationships](#7-relationships)
- [8. Navigation](#8-navigation)
- [9. Widgets and the dashboard](#9-widgets-and-the-dashboard)
- [10. Custom pages](#10-custom-pages)
- [11. Who's allowed in](#11-whos-allowed-in)
  - [Access to the panel](#access-to-the-panel)
  - [Access to records](#access-to-records)
- [12. A second panel](#12-a-second-panel)
- [13. Theming](#13-theming)
- [14. Coming from v3 or v4](#14-coming-from-v3-or-v4)
- [15. Deploying](#15-deploying)
- [16. When it breaks](#16-when-it-breaks)
- [17. Command reference](#17-command-reference)

---

## 1. What you need first

Filament v5 needs:

| Requirement | Version |
| --- | --- |
| PHP | 8.2+ |
| Laravel | v11.28+ |
| Tailwind CSS | v4.1+ |

Tailwind only matters if you write your own views or a custom theme — the panel
ships with its own compiled CSS, so a stock install needs no npm step at all.

Starting from nothing:

```bash
composer create-project laravel/laravel my-app
cd my-app
```

Set up the database in `.env` and run `php artisan migrate` before installing
Filament. The installer doesn't need the database, but `make:filament-user` does,
and it's the next thing you'll want.

---

## 2. Install Filament

Two commands:

```bash
composer require filament/filament:"^5.0"
php artisan filament:install --panels
```

On Windows PowerShell use `"~5.0"` instead — PowerShell eats the `^`.

Then create an account to log in with:

```bash
php artisan make:filament-user
```

It asks for a name, email and password, and writes a row to `users`. Open
`/admin`, sign in, and you're looking at an empty dashboard.

If you want to change Filament's global defaults later — default filesystem disk,
file generation flags, UI defaults — publish the config:

```bash
php artisan vendor:publish --tag=filament-config
```

That creates `config/filament.php`. You don't need it to get started.

### Running the commands in Docker

In the containerised setup, nothing PHP-side runs on the host. Prefix every
command in this guide:

```bash
docker compose exec php composer require filament/filament:"^5.0"
docker compose exec php php artisan filament:install --panels
docker compose exec php php artisan make:filament-user
```

The rest of this guide writes them bare for readability.

---

## 3. What the installer created

One file, and one line in another.

`app/Providers/Filament/AdminPanelProvider.php` is the whole configuration of the
panel. It's a Laravel service provider, and Filament registers it in
`bootstrap/providers.php` for you:

```php
public function panel(Panel $panel): Panel
{
    return $panel
        ->default()
        ->id('admin')
        ->path('admin')
        ->login()
        ->colors([
            'primary' => Color::Amber,
        ])
        ->discoverResources(in: app_path('Filament/Resources'), for: 'App\Filament\Resources')
        ->discoverPages(in: app_path('Filament/Pages'), for: 'App\Filament\Pages')
        ->pages([
            Dashboard::class,
        ])
        ->discoverWidgets(in: app_path('Filament/Widgets'), for: 'App\Filament\Widgets')
        ->widgets([
            AccountWidget::class,
            FilamentInfoWidget::class,
        ])
        ->middleware([
            EncryptCookies::class,
            AddQueuedCookiesToResponse::class,
            StartSession::class,
            AuthenticateSession::class,
            ShareErrorsFromSession::class,
            PreventRequestForgery::class,
            SubstituteBindings::class,
            DisableBladeIconComponents::class,
            DispatchServingFilamentEvent::class,
        ])
        ->authMiddleware([
            Authenticate::class,
        ]);
}
```

The lines worth knowing on day one:

| Method | What it does |
| --- | --- |
| `id()` | The panel's unique name. Used by `getUrl(panel: ...)` and `canAccessPanel()` |
| `path()` | Where it lives — `admin` gives you `/admin`. `''` puts it at the root |
| `default()` | Only one panel gets this; it's the fallback when nothing else matches |
| `login()` | Registers the login page. Drop it and there's no way in |
| `discoverResources()` | Auto-registers everything in `app/Filament/Resources`, so you rarely edit this file after creating a resource |
| `colors()` | The primary colour, from `Filament\Support\Colors\Color` |

If `/admin` 404s, the provider isn't registered — check `bootstrap/providers.php`.

---

## 4. Your first resource

A **resource** is a CRUD interface for one Eloquent model: a list page, a create
page, an edit page, and optionally a view page.

### The model and migration

Filament reads your database columns to scaffold, so build the model first:

```bash
php artisan make:model Post -m
```

```php
// database/migrations/xxxx_xx_xx_create_posts_table.php
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->string('slug')->unique();
    $table->text('body')->nullable();
    $table->string('status')->default('draft');
    $table->boolean('is_featured')->default(false);
    $table->timestamp('published_at')->nullable();
    $table->timestamps();
});
```

```php
// app/Models/Post.php
protected $fillable = [
    'title', 'slug', 'body', 'status', 'is_featured', 'published_at',
];
```

```bash
php artisan migrate
```

`$fillable` matters. A field that isn't fillable will render, accept input, and
silently not save.

### Generating the resource

```bash
php artisan make:filament-resource Post --generate
```

`--generate` reads the table and writes a form and table for you. It's the
fastest way to get something on screen, and you edit it afterwards.

The other flags:

| Flag | What you get |
| --- | --- |
| `--generate` | Form and table built from the database columns |
| `--simple` | One "Manage" page, create and edit in modals. No relation managers |
| `--view` | An extra read-only View page |
| `--soft-deletes` | Restore, force-delete, and a trashed filter |
| `--model --migration --factory` | Generates those alongside the resource |
| `--model-namespace=` | If your models don't live in `App\Models` |

Refresh `/admin` — "Posts" is in the sidebar already. Nothing needed registering,
because `discoverResources()` found it.

### What each generated file does

```
app/Filament/Resources/
└── Posts/
    ├── PostResource.php          the resource: model, icon, wiring
    ├── Pages/
    │   ├── CreatePost.php        the create page (a Livewire component)
    │   ├── EditPost.php          the edit page
    │   └── ListPosts.php         the list page
    ├── Schemas/
    │   └── PostForm.php          the form definition
    └── Tables/
        └── PostsTable.php        the table definition
```

Note the nesting: v5 puts each resource in its own directory. `PostResource.php`
is deliberately thin — it delegates:

```php
use App\Filament\Resources\Posts\Schemas\PostForm;
use App\Filament\Resources\Posts\Tables\PostsTable;
use Filament\Schemas\Schema;
use Filament\Tables\Table;

public static function form(Schema $schema): Schema
{
    return PostForm::configure($schema);
}

public static function table(Table $table): Table
{
    return PostsTable::configure($table);
}
```

You can inline both and delete the extra classes — the docs explicitly allow it.
The split exists because resource classes get long fast.

---

## 5. The form

Forms are **schemas**: `Filament\Schemas\Schema`, a list of components rendered
in order.

```php
// app/Filament/Resources/Posts/Schemas/PostForm.php
use Filament\Forms\Components\DateTimePicker;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Textarea;
use Filament\Forms\Components\Toggle;
use Filament\Schemas\Schema;

public static function configure(Schema $schema): Schema
{
    return $schema
        ->components([
            TextInput::make('title')
                ->required()
                ->maxLength(255),
            TextInput::make('slug')
                ->required()
                ->unique(),
            Textarea::make('body')
                ->rows(10)
                ->columnSpanFull(),
            Select::make('status')
                ->options([
                    'draft' => 'Draft',
                    'published' => 'Published',
                ])
                ->default('draft')
                ->required(),
            Toggle::make('is_featured'),
            DateTimePicker::make('published_at'),
        ]);
}
```

Field classes live in `Filament\Forms\Components\` — `TextInput`, `Select`,
`Textarea`, `Checkbox`, `Toggle`, `DateTimePicker`, `FileUpload`, `RichEditor`,
`Repeater`, and so on. The name you pass to `make()` is the model attribute.

Validation is chained onto the field (`->required()`, `->email()`,
`->maxLength()`, `->unique()`), not declared in a separate rules array.

Inside a resource, `->unique()` already ignores the record being edited — you
don't need `ignoreRecord: true` the way you did in v3. Pass
`->unique(ignoreRecord: false)` if you actually want it to collide with itself.

### Layout components

Layout comes from `Filament\Schemas\Components\` — a different namespace from the
fields, which trips people up:

```php
use Filament\Schemas\Components\Grid;
use Filament\Schemas\Components\Section;

Section::make('Content')
    ->description('The bits readers see')
    ->schema([
        TextInput::make('title')->required(),
        Textarea::make('body'),
    ]),

Grid::make(2)
    ->schema([
        Select::make('status')->options([/* ... */]),
        DateTimePicker::make('published_at'),
    ]),
```

`Section`, `Grid`, `Tabs` and `Wizard` all live there. `Wizard` turns a long form
into steps without changing any of the fields.

### Showing a field on only one page

A password field belongs on create, not on edit:

```php
use Filament\Support\Enums\Operation;

TextInput::make('password')
    ->password()
    ->required()
    ->visibleOn(Operation::Create),
```

`hiddenOn()` is the inverse.

---

## 6. The table

```php
// app/Filament/Resources/Posts/Tables/PostsTable.php
use Filament\Actions\BulkActionGroup;
use Filament\Actions\DeleteBulkAction;
use Filament\Actions\EditAction;
use Filament\Tables\Columns\IconColumn;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Filters\SelectFilter;
use Filament\Tables\Table;

public static function configure(Table $table): Table
{
    return $table
        ->columns([
            TextColumn::make('title')
                ->searchable()
                ->sortable(),
            TextColumn::make('status')
                ->badge(),
            IconColumn::make('is_featured')
                ->boolean(),
            TextColumn::make('published_at')
                ->dateTime()
                ->sortable()
                ->toggleable(),
        ])
        ->filters([
            SelectFilter::make('status')
                ->options([
                    'draft' => 'Draft',
                    'published' => 'Published',
                ]),
        ])
        ->recordActions([
            EditAction::make(),
        ])
        ->toolbarActions([
            BulkActionGroup::make([
                DeleteBulkAction::make(),
            ]),
        ]);
}
```

Three things to know:

- Columns are `Filament\Tables\Columns\` — `TextColumn`, `IconColumn`,
  `ImageColumn`, `ColorColumn`, `SelectColumn`, `ToggleColumn`.
- **Actions are `Filament\Actions\`**, not `Filament\Tables\Actions\`. Every
  action class in v5 lives in one namespace, whether it's on a table row, a page
  header, or inside a form.
- `->recordActions()` is the per-row buttons, `->toolbarActions()` is the header
  and bulk buttons.

Dot notation reaches through relationships: `TextColumn::make('author.name')`.
`->searchable()` on that column searches the joined table.

---

## 7. Relationships

A **relation manager** is a table of related records embedded in the parent's
Edit or View page — comments under a post, addresses under a customer.

```bash
php artisan make:filament-relation-manager PostResource comments body
```

Three arguments: the owner's resource class, the relationship method on the
model, and the attribute that identifies a related record.

That writes `PostResource/RelationManagers/CommentsRelationManager.php`, which
has its own `form()` and `table()` — the same API as the resource, just
non-static.

Register it on the resource:

```php
public static function getRelations(): array
{
    return [
        RelationManagers\CommentsRelationManager::class,
    ];
}
```

Useful flags: `--attach` and `--associate` add buttons for linking existing
records (many-to-many and one-to-many respectively), `--view` adds a view modal,
`--soft-deletes` adds trashed handling.

On a View page, relation managers go read-only automatically. Override
`isReadOnly()` to return `false` if you want them editable there.

Simple (`--simple`) resources have no `getRelations()`, because relation managers
only render on Edit and View pages, and a simple resource has neither.

---

## 8. Navigation

Sidebar entries come from properties on the resource class:

```php
use BackedEnum;
use UnitEnum;

protected static string | BackedEnum | null $navigationIcon = 'heroicon-o-document-text';

protected static string | UnitEnum | null $navigationGroup = 'Content';

protected static ?string $navigationLabel = 'Blog posts';

protected static ?int $navigationSort = 2;
```

Icons are [Heroicons](https://heroicons.com) by default — any Blade icon
component name works.

Two more worth setting early:

```php
protected static ?string $recordTitleAttribute = 'title';  // needed for global search
protected static ?string $slug = 'blog-posts';             // changes /admin/posts to /admin/blog-posts
```

Never hand-build URLs to resource pages. Use `getUrl()`:

```php
PostResource::getUrl();                              // /admin/posts
PostResource::getUrl('create');                      // /admin/posts/create
PostResource::getUrl('edit', ['record' => $post]);   // /admin/posts/edit/1
```

---

## 9. Widgets and the dashboard

```bash
php artisan make:filament-widget StatsOverview --stats-overview
```

The bare `make:filament-widget MyWidget` asks which type you want: custom, chart,
stats overview, or table. The flags skip the prompt.

```php
// app/Filament/Widgets/StatsOverview.php
use Filament\Widgets\StatsOverviewWidget as BaseWidget;
use Filament\Widgets\StatsOverviewWidget\Stat;

class StatsOverview extends BaseWidget
{
    protected function getStats(): array
    {
        return [
            Stat::make('Posts', Post::count()),
            Stat::make('Published', Post::where('status', 'published')->count()),
            Stat::make('Featured', Post::where('is_featured', true)->count()),
        ];
    }
}
```

`discoverWidgets()` picks it up, so it appears on the dashboard without
registration. Order them with `protected static ?int $sort = 2;`.

Every widget is a Livewire component, so anything Livewire can do — polling,
events, actions — a widget can do.

To clear out the two default widgets, remove `AccountWidget` and
`FilamentInfoWidget` from the `widgets()` array in the panel provider.

---

## 10. Custom pages

When something isn't CRUD — a settings screen, a report, an import wizard:

```bash
php artisan make:filament-page Settings
```

You get `app/Filament/Pages/Settings.php` and a Blade view at
`resources/views/filament/pages/settings.blade.php`. The class is a full-page
Livewire component, so the view is yours to fill.

Pages take the same navigation properties as resources, and can host header
actions and widgets. `discoverPages()` registers them automatically.

---

## 11. Who's allowed in

Two separate layers. Getting this wrong is the single most common way a Filament
app breaks on deploy and works fine locally.

### Access to the panel

Locally, every `App\Models\User` can sign in. The moment `APP_ENV` isn't `local`,
Filament requires your user model to implement `FilamentUser` — and if it
doesn't, **nobody can sign in at all**:

```php
use Filament\Models\Contracts\FilamentUser;
use Filament\Panel;

class User extends Authenticatable implements FilamentUser
{
    public function canAccessPanel(Panel $panel): bool
    {
        return str_ends_with($this->email, '@example.edu') && $this->hasVerifiedEmail();
    }
}
```

You get the `$panel`, so you can answer differently per panel:

```php
public function canAccessPanel(Panel $panel): bool
{
    if ($panel->getId() === 'admin') {
        return $this->is_admin;
    }

    return true;
}
```

### Access to records

Filament respects ordinary Laravel model policies. No extra API:

```bash
php artisan make:policy PostPolicy --model=Post
```

The methods it uses: `viewAny()`, `view()`, `create()`, `update()`, `delete()`
and `deleteAny()`, plus `forceDelete()`/`forceDeleteAny()`,
`restore()`/`restoreAny()`, and `reorder()`.

`viewAny()` is the one to remember — return `false` and the resource vanishes
from the navigation entirely. If you just created a resource and it isn't in the
sidebar, check the policy first.

Bulk actions check `deleteAny()` rather than looping `delete()`, for performance.
If you need the per-record check anyway, use
`DeleteBulkAction::make()->authorizeIndividualRecords()`.

---

## 12. A second panel

The `/admin` and `/app` split — staff in one, end users in the other — is one
Laravel app with two panels:

```bash
php artisan make:filament-panel app
```

That writes `app/Providers/Filament/AppPanelProvider.php`, serving `/app`, and
discovering its own components in `app/Filament/App/Resources`, `.../Pages`,
`.../Widgets`. The two panels share models, migrations and users, and nothing
else.

Only one panel may call `->default()`. Which resources a user sees is settled by
`canAccessPanel()` reading `$panel->getId()`.

---

## 13. Theming

Colours and font are panel configuration, no build step:

```php
use Filament\Support\Colors\Color;

->colors([
    'primary' => Color::Indigo,
    'danger' => Color::Rose,
    'gray' => Color::Gray,
    'info' => Color::Blue,
    'success' => Color::Emerald,
    'warning' => Color::Orange,
])
->font('Figtree')
```

`Color` covers the Tailwind palettes. A hex or `rgb()` string works too — Filament
generates the shades.

For actual CSS changes, or to use Tailwind classes in your own Filament views,
you need a custom theme:

```bash
php artisan make:filament-theme          # or: make:filament-theme admin
```

It installs the Tailwind dependencies, writes
`resources/css/filament/admin/theme.css`, adds it to `vite.config.js`, and
registers `->viteTheme()` on the panel. If your files are formatted unusually it
prints manual steps instead of guessing.

The generated theme has `@source` lines telling Tailwind where to scan. Add your
own directories or your classes get stripped:

```css
@source '../../../../app/Filament/**/*';
@source '../../../../resources/views/filament/**/*';
@source '../../../../resources/views/livewire/**/*';
```

Then `npm run build`.

---

## 14. Coming from v3 or v4

Most stale-tutorial confusion is one of these:

| Then | Now (v5) |
| --- | --- |
| `app/Filament/Resources/PostResource.php` | `app/Filament/Resources/Posts/PostResource.php` |
| Form and table inline in the resource | Split into `Schemas/PostForm.php` and `Tables/PostsTable.php` |
| `form(Form $form): Form` | `form(Schema $schema): Schema` |
| `Filament\Forms\Form` | `Filament\Schemas\Schema` |
| `Filament\Forms\Components\Section`, `Grid`, `Tabs`, `Wizard` | `Filament\Schemas\Components\...` |
| `Filament\Tables\Actions\EditAction` | `Filament\Actions\EditAction` |
| `->actions([...])` | `->recordActions([...])` |
| `->bulkActions([...])` | `->toolbarActions([...])` |
| Separate `infolist()` API | Infolists are schemas too, same `->components()` |
| `->unique(ignoreRecord: true)` | `->unique()` — ignoring the current record is the default |

If a snippet doesn't compile, check the namespace before anything else — the
class usually still exists, somewhere else.

---

## 15. Deploying

Three things beyond a normal Laravel deploy.

**Authorize your users.** Covered in section 11 — without `FilamentUser`, a
production panel locks everyone out. This is deliberate.

**Cache the components.** Filament scans directories to discover resources,
pages and widgets. Cache that:

```bash
php artisan filament:optimize
```

That's `filament:cache-components` plus `icons:cache` in one. Run
`php artisan optimize` for Laravel's own config and route caches while you're
there.

Don't run these locally — new components stop being discovered until you clear
the cache:

```bash
php artisan filament:optimize-clear
```

**Leave `filament:upgrade` alone.** The installer added it to the
`post-autoload-dump` script in `composer.json`. It republishes assets after every
package update. Remove it and you get subtly stale CSS and JS in production.

Also worth checking: OPcache is on.

```bash
php -r "echo 'opcache.enable => ' . ini_get('opcache.enable') . PHP_EOL;"
```

---

## 16. When it breaks

| What you see | What it usually is | What to do |
| --- | --- | --- |
| `/admin` returns 404 | Panel provider not registered | Check `bootstrap/providers.php` for `AdminPanelProvider` |
| Nobody can sign in after deploy | `APP_ENV` isn't `local` and `User` doesn't implement `FilamentUser` | See section 11 |
| Resource missing from the sidebar | Policy `viewAny()` returns `false`, or the component cache is stale | Fix the policy, or `php artisan filament:clear-cached-components` |
| New resource isn't found at all | Namespace doesn't match the `discover*()` path, or components are cached | Check the class namespace against the panel provider |
| `Class "Filament\Forms\Form" not found` | v3/v4 snippet | See section 14 |
| Field renders but never saves | Attribute missing from the model's `$fillable` | Add it |
| Tailwind classes in a custom view do nothing | No custom theme, or the directory isn't in `@source` | See section 13 |
| Blank page, nothing in the log | A binary column (geometry, blob) is being serialized to the browser | Add it to the model's `$hidden` |
| Styles broken after a package update | Assets not republished | `php artisan filament:upgrade` |
| Changes don't show up locally | Component cache left on | `php artisan filament:optimize-clear` |

---

## 17. Command reference

```bash
# install
composer require filament/filament:"^5.0"
php artisan filament:install --panels
php artisan make:filament-user
php artisan vendor:publish --tag=filament-config

# build
php artisan make:filament-resource Post --generate
php artisan make:filament-resource Post --simple --view --soft-deletes
php artisan make:filament-relation-manager PostResource comments body
php artisan make:filament-widget StatsOverview --stats-overview
php artisan make:filament-page Settings
php artisan make:filament-panel app
php artisan make:filament-theme

# maintain
php artisan filament:optimize          # cache components + icons, for deploys
php artisan filament:optimize-clear    # undo the above
php artisan filament:upgrade           # republish assets after an update
```
