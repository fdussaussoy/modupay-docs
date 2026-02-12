# modupay-docs

Documentation officielle de l'API Modupay — publiée sur GitHub Pages avec Jekyll + just-the-docs.

🌐 **Site en ligne** : https://VOTRE_USERNAME.github.io/modupay-docs

---

## Structure

```
modupay-docs/
├── index.md                    # Page d'accueil
├── _config.yml                 # Config Jekyll
├── Gemfile                     # Dépendances Ruby
├── _sass/color_schemes/        # Thème couleurs Modupay
├── .github/workflows/          # CI/CD auto-deploy
└── docs/
    ├── overview.md             # Vue d'ensemble & architecture
    ├── integration-guide.md    # Guide d'intégration
    ├── technical-architecture.md # Architecture technique
    ├── api-reference.md        # Référence des endpoints
    ├── practical-examples.md   # Exemples de code
    └── payment-api-spec.yaml   # Spécification OpenAPI
```

## Mise en ligne (5 min)

### 1. Créer le repo GitHub

```bash
git init
git add .
git commit -m "Initial documentation Modupay V1"
git branch -M main
git remote add origin https://github.com/VOTRE_USERNAME/modupay-docs.git
git push -u origin main
```

### 2. Activer GitHub Pages

1. Aller dans **Settings** → **Pages**
2. Source : **GitHub Actions**
3. Le site se déploie automatiquement ✅

### 3. Mettre à jour `_config.yml`

Remplacer `VOTRE_USERNAME` par votre username GitHub :

```yaml
url: "https://VOTRE_USERNAME.github.io"
```

---

## Développement local (optionnel)

```bash
gem install bundler
bundle install
bundle exec jekyll serve
# → http://localhost:4000
```

---

## Mise à jour du contenu

Chaque `git push` sur `main` redéploie automatiquement le site via GitHub Actions.

---

© 2026 Modupay SAS
