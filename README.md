# Devise API Authentication | Ruby on Rails 7 Tutorial

Date: February 28, 2022 4:01 PM
Tags: Authentification, Backend, Dev, Devise, Learning, Rails, Video, Web, Youtube
URL: https://www.youtube.com/watch?v=PqizV5l1yFE

---

## Création de l’API 🛤

### Création d’une app Rails en mode API *(donc sans front)*

`rails new my_api --api`

### Rajout de 3 gems : devise, devise-jwt, et rack-cors

`bundle add devise devise-jwt rack-cors`

- Devise sert au setup de tout le système d’authentification en tant que tel
- Devise-jwt est une extension de Devise permettant d’utiliser les JWT token pour l’authentification
- Rack CORS

### Configuration de Rack CORS

C’est parti pour quelques modifications dans le fichier `config/initializers/cors.rb`

Ces changements permettent d’autoriser n’importe quel site à faire des requêtes à l’API, pour autoriser une seule origine ➡️ `origins "[url]"`

```ruby
# config/initializers/cors.rb

# Be sure to restart your server when you modify this file.

# Avoid CORS issues when API is called from the frontend app.
# Handle Cross-Origin Resource Sharing (CORS) in order to accept cross-origin AJAX requests.

# Read more: https://github.com/cyu/rack-cors

Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins '*'

    resource '*',
             headers: :any,
             methods: %i[get post put patch delete options head],
             expose: %w[Authorization Uid]
  end
end
```

### Installation de Devise et génération de la table User

`rails g devise:install`

`rails g devise User`

## Devise JWT 💲

### Génération de la Denylist *(sert pour la sécurité)*

`rails g model jwt_denylist jti:string exp:datetime`

- Le jti est l’identifiant unique d’un token
- Exp contient sa date d’expiration

<aside>
⚠️ Pour que tout fonctionne, vous devez renommer le fichier de migration (de `[timestamp]_create_jwt_denylists.rb` à `[timestamp]_create_jwt_denylist.rb`), la classe et la table au singulier (voir en-dessous)

</aside>

Fichier de migration :

```ruby
# db/migrate/20220228223034_create_jwt_denylist.rb

class CreateJwtDenylist < ActiveRecord::Migration[7.0]
  def change
    create_table :jwt_denylist do |t|
      t.string :jti, null: false
      t.datetime :exp, null: false

      t.timestamps
    end
    add_index :jwt_denylist, :jti
  end
end
```

### Petit changements au modèle User

- `:jwt_authenticatable` permet de dire à Devise que `User` utilise jwt pour l’authentification
- `:jwt_revocation_strategy` permet de dire à `User` comment il doit révoquer les tokens, et qu’il doit utiliser le modèle `JwtDenylist` pour ça

```ruby
# app/models/user.rb

class User < ApplicationRecord
	# Il faut ajouter les deux modules commençant par jwt
	devise :database_authenticatable, :registerable,
	:jwt_authenticatable,
	jwt_revocation_strategy: JwtDenylist
end
```

### Petits ajouts des familles au Model `JwtDenylist`

On indique aussi au modèle `JwtDenylist` qu’il doit utiliser la stratégie de révocation `denylist` (oui oui)

```ruby
# app/models/jwt_denylist.rb

class JwtDenylist < ApplicationRecord
  include Devise::JWT::RevocationStrategies::Denylist

  self.table_name = 'jwt_denylist'
end
```

### `rails db:migrate` 🙂

## Devise API JWT Controllers for Sessions and Registrations 🧒

### Créer le fichier `members_controller.rb`

La méthode `show` permettra de s’authentifier avec un token au lieu d’avec l’email et le password

```ruby
# app/controllers/members_controller.rb

class MembersController < ApplicationController
  before_action :authenticate_user!

  def show
    user = get_user_from_token
    render json: {
      message: "If you see this, you're in!",
      user: user
    }
  end

  private

  def get_user_from_token
    jwt_payload = JWT.decode(request.headers['Authorization'].split(' ')[1],
                             Rails.application.credentials.devise[:jwt_secret_key]).first
    user_id = jwt_payload['sub']
    User.find(user_id.to_s)
  end
end
```

### Création des Users

Deux nouveaux controllers à créer, qui modifieront les controllers de registration et de session de Devise

<aside>
⚠️ Il faut créer ces fichiers dans un dossier `users` dans `app/controllers` (voir le commentaire en haut des snippets)

</aside>

```ruby
# app/controllers/users/registrations_controller.rb

class Users::RegistrationsController < Devise::RegistrationsController
  respond_to :json

  private

  def respond_with(resource, _opts = {})
    register_success && return if resource.persisted?

    register_failed
  end

  def register_success
    render json: {
      message: 'Signed up sucessfully.',
      user: current_user
    }, status: :ok
  end

  def register_failed
    render json: { message: 'Something went wrong.' }, status: :unprocessable_entity
  end
end
```

```ruby
# app/controllers/users/sessions_controller.rb

class Users::SessionsController < Devise::SessionsController
  respond_to :json

  private

  def respond_with(_resource, _opts = {})
    render json: {
      message: 'You are logged in.',
      user: current_user
    }, status: :ok
  end

  def respond_to_on_destroy
    log_out_success && return if current_user

    log_out_failure
  end

  def log_out_success
    render json: { message: 'You are logged out.' }, status: :ok
  end

  def log_out_failure
    render json: { message: 'Hmm nothing happened.' }, status: :unauthorized
  end
end
```

## Devise JWT Secret Key 🔑

### Config du secret JWT utilisé pour décoder les tokens dans `config/initializers/devise.rb`

```ruby
Devise.setup do |config|
	# Plein de code
	config.jwt do |jwt|
		jwt.secret = Rails.application.credentials.devise[:jwt_secret_key]
	end
	# Encore tout plein de code
end
```

1. Génération du secret
    - `rake secret`
    - Copie de la string générée
    - `EDITOR=nano rails credentials:edit`
    - Ajout en bas du fichier de :
    
    ```bash
    devise:
      jwt_secret_key: [clé copiée] // ⚠ Il faut mettre 2 espaces au début de cette ligne
    ```
    

## Routes 🛣

### Go `config/routes.rb`

```ruby
# config/routes.rb

Rails.application.routes.draw do
  devise_for :users,
             controllers: {
               sessions: 'users/sessions',
               registrations: 'users/registrations'
             }
  get '/member-data', to: 'members#show'
  # Define your application routes per the DSL in https://guides.rubyonrails.org/routing.html

  # Defines the root path route ("/")
  # root "articles#index"
end
```

## Configure Session Store in Rails 7

1. Config pour utiliser les cookies dans `config/application.rb`

```ruby
# config/application.rb

module DeviseVue
  class Application < Rails::Application
    # Du code cool

    # This also configures session_options for use below
    config.session_store :cookie_store, key: '_interslice_session'

    # Required for all session management (regardless of session_store)
    config.middleware.use ActionDispatch::Cookies

    config.middleware.use config.session_store, config.session_options

    # Plein de code
  end
end
```

Et voilà ! 🎉

## Et les routes c’est quoi ?

### Register

`POST /users`

Données attendues :

```json
{
	"user": {
		"email": string,
		"password": string
	}
}
```

### Login

`POST /users/sign_in`

Données attendues

```json
{
	"user": {
		"email": string,
		"password": string
	}
}
```

### Logout

`DELETE /users/sign_out`

Authentification nécessaire

### Login with token

`GET /member-data`

Authentification nécessaire