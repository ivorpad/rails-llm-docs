# API Authentication Best Practices

## Core Principle
Use the simplest authentication system that meets your needs. Start with API keys, move to tokens if needed, and only use OAuth when required.

## Simple API Key Authentication

### Basic API Key Implementation
```ruby
# Migration
class AddApiKeyToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :api_key, :string
    add_index :users, :api_key, unique: true
  end
end

# Model
class User < ApplicationRecord
  before_create :generate_api_key
  
  def regenerate_api_key!
    generate_api_key
    save!
  end
  
  private
  
  def generate_api_key
    self.api_key = SecureRandom.hex(32)
  end
end

# Controller
class Api::V1::BaseController < ActionController::API
  before_action :authenticate!
  
  private
  
  def authenticate!
    api_key = request.headers['X-API-Key'] || params[:api_key]
    
    @current_user = User.find_by(api_key: api_key)
    
    unless @current_user
      render json: { error: 'Invalid API key' }, status: :unauthorized
    end
  end
  
  attr_reader :current_user
end
```

### API Key with Rate Limiting
```ruby
# Using Redis for rate limiting
class Api::V1::BaseController < ActionController::API
  before_action :authenticate!
  before_action :check_rate_limit!
  
  private
  
  def check_rate_limit!
    key = "api_rate_limit:#{current_user.id}"
    count = Redis.current.incr(key)
    
    if count == 1
      Redis.current.expire(key, 1.hour)
    end
    
    if count > rate_limit
      render json: { 
        error: 'Rate limit exceeded',
        retry_after: Redis.current.ttl(key)
      }, status: :too_many_requests
    end
  end
  
  def rate_limit
    current_user.premium? ? 10000 : 1000
  end
end
```

## Token-Based Authentication (JWT)

### JWT Implementation
```ruby
# Gemfile
gem 'jwt'

# app/services/json_web_token.rb
class JsonWebToken
  SECRET_KEY = Rails.application.credentials.secret_key_base
  
  def self.encode(payload, exp = 24.hours.from_now)
    payload[:exp] = exp.to_i
    JWT.encode(payload, SECRET_KEY)
  end
  
  def self.decode(token)
    decoded = JWT.decode(token, SECRET_KEY)[0]
    HashWithIndifferentAccess.new(decoded)
  rescue JWT::DecodeError => e
    nil
  end
end

# Authentication controller
class Api::V1::AuthController < Api::V1::BaseController
  skip_before_action :authenticate!
  
  def login
    user = User.find_by(email: params[:email])
    
    if user&.authenticate(params[:password])
      token = JsonWebToken.encode(user_id: user.id)
      
      render json: {
        token: token,
        expires_at: 24.hours.from_now,
        user: user.as_json(only: [:id, :email, :name])
      }
    else
      render json: { error: 'Invalid credentials' }, status: :unauthorized
    end
  end
  
  def refresh
    if current_user
      token = JsonWebToken.encode(user_id: current_user.id)
      render json: { token: token, expires_at: 24.hours.from_now }
    else
      render json: { error: 'Invalid token' }, status: :unauthorized
    end
  end
end

# Base controller with JWT auth
class Api::V1::BaseController < ActionController::API
  before_action :authenticate!
  
  private
  
  def authenticate!
    header = request.headers['Authorization']
    header = header.split(' ').last if header
    
    decoded = JsonWebToken.decode(header)
    @current_user = User.find(decoded[:user_id]) if decoded
  rescue ActiveRecord::RecordNotFound
    render json: { error: 'Unauthorized' }, status: :unauthorized
  end
  
  attr_reader :current_user
end
```

## Bearer Token with Expiry

### Database-Backed Tokens
```ruby
# Migration
class CreateApiTokens < ActiveRecord::Migration[7.0]
  def change
    create_table :api_tokens do |t|
      t.references :user, null: false, foreign_key: true
      t.string :token, null: false
      t.string :name
      t.datetime :last_used_at
      t.datetime :expires_at
      t.timestamps
    end
    
    add_index :api_tokens, :token, unique: true
    add_index :api_tokens, :expires_at
  end
end

# Model
class ApiToken < ApplicationRecord
  belongs_to :user
  
  before_create :generate_token
  
  scope :active, -> { where('expires_at > ? OR expires_at IS NULL', Time.current) }
  
  def expired?
    expires_at.present? && expires_at < Time.current
  end
  
  def record_usage!
    update_column(:last_used_at, Time.current)
  end
  
  private
  
  def generate_token
    self.token = SecureRandom.urlsafe_base64(32)
  end
end

# Controller
class Api::V1::BaseController < ActionController::API
  before_action :authenticate!
  
  private
  
  def authenticate!
    bearer = request.headers['Authorization']
    token = bearer.split(' ').last if bearer&.starts_with?('Bearer ')
    
    api_token = ApiToken.active.find_by(token: token)
    
    if api_token
      api_token.record_usage!
      @current_user = api_token.user
    else
      render json: { error: 'Invalid or expired token' }, status: :unauthorized
    end
  end
end
```

## OAuth2 Implementation (When Needed)

### Using Doorkeeper for OAuth2
```ruby
# Gemfile
gem 'doorkeeper'

# config/initializers/doorkeeper.rb
Doorkeeper.configure do
  orm :active_record
  
  resource_owner_authenticator do
    current_user || redirect_to(login_url)
  end
  
  admin_authenticator do
    current_user&.admin? || redirect_to(root_url)
  end
  
  authorization_code_expires_in 10.minutes
  access_token_expires_in 2.hours
  
  grant_flows %w[authorization_code client_credentials password]
  
  skip_authorization do |resource_owner, client|
    client.trusted?
  end
end

# Routes
Rails.application.routes.draw do
  use_doorkeeper
  
  namespace :api do
    namespace :v1 do
      resources :users
    end
  end
end

# Controller
class Api::V1::BaseController < ActionController::API
  before_action :doorkeeper_authorize!
  
  private
  
  def current_user
    @current_user ||= User.find(doorkeeper_token.resource_owner_id) if doorkeeper_token
  end
end
```

## Service-to-Service Authentication

### HMAC Signature Authentication
```ruby
class Api::V1::WebhookController < ActionController::API
  skip_before_action :verify_authenticity_token
  before_action :verify_signature!
  
  def receive
    # Process webhook
    render json: { status: 'received' }
  end
  
  private
  
  def verify_signature!
    secret = Rails.application.credentials.webhook_secret
    signature = request.headers['X-Webhook-Signature']
    
    expected = OpenSSL::HMAC.hexdigest(
      'SHA256',
      secret,
      request.raw_post
    )
    
    unless ActiveSupport::SecurityUtils.secure_compare(signature, expected)
      render json: { error: 'Invalid signature' }, status: :unauthorized
    end
  end
end
```

## Multi-Tenant API Authentication

### Scoped API Keys
```ruby
class Organization < ApplicationRecord
  has_many :api_keys
  has_many :users
end

class ApiKey < ApplicationRecord
  belongs_to :organization
  belongs_to :created_by, class_name: 'User'
  
  enum role: { read_only: 0, read_write: 1, admin: 2 }
  
  before_create :generate_key
  
  private
  
  def generate_key
    self.key = "#{organization.slug}_#{SecureRandom.hex(24)}"
  end
end

class Api::V1::BaseController < ActionController::API
  before_action :authenticate!
  before_action :set_current_organization!
  
  private
  
  def authenticate!
    api_key = request.headers['X-API-Key']
    @api_key_record = ApiKey.find_by(key: api_key)
    
    unless @api_key_record
      render json: { error: 'Invalid API key' }, status: :unauthorized
    end
  end
  
  def set_current_organization!
    @current_organization = @api_key_record.organization
  end
  
  def authorize!(action, resource)
    case action
    when :read
      true
    when :write
      @api_key_record.read_write? || @api_key_record.admin?
    when :admin
      @api_key_record.admin?
    else
      false
    end
  end
  
  attr_reader :current_organization
end
```

## Testing API Authentication

### RSpec Request Specs
```ruby
RSpec.describe "API Authentication" do
  let(:user) { create(:user) }
  
  describe "API Key authentication" do
    it "allows access with valid API key" do
      get "/api/v1/products", 
          headers: { "X-API-Key" => user.api_key }
      
      expect(response).to have_http_status(:ok)
    end
    
    it "denies access without API key" do
      get "/api/v1/products"
      
      expect(response).to have_http_status(:unauthorized)
      expect(JSON.parse(response.body)['error']).to eq('Invalid API key')
    end
  end
  
  describe "JWT authentication" do
    let(:token) { JsonWebToken.encode(user_id: user.id) }
    
    it "allows access with valid token" do
      get "/api/v1/products",
          headers: { "Authorization" => "Bearer #{token}" }
      
      expect(response).to have_http_status(:ok)
    end
    
    it "denies access with expired token" do
      expired_token = JsonWebToken.encode(
        { user_id: user.id }, 
        1.hour.ago
      )
      
      get "/api/v1/products",
          headers: { "Authorization" => "Bearer #{expired_token}" }
      
      expect(response).to have_http_status(:unauthorized)
    end
  end
end
```

## Security Best Practices

### Secure Token Storage
```ruby
# Never log tokens
class Api::V1::BaseController < ActionController::API
  # Filter tokens from logs
  filter_parameter_logging :token, :api_key, :password
  
  before_action :authenticate!
  
  private
  
  def authenticate!
    # Use constant-time comparison
    api_key = request.headers['X-API-Key']
    user = User.find_by(api_key: api_key)
    
    if user && ActiveSupport::SecurityUtils.secure_compare(
      user.api_key,
      api_key
    )
      @current_user = user
    else
      render json: { error: 'Unauthorized' }, status: :unauthorized
    end
  end
end
```

### Token Rotation
```ruby
class RotateApiKeys
  def call
    User.find_each do |user|
      old_key = user.api_key
      user.regenerate_api_key!
      
      # Notify user
      ApiKeyMailer.rotation_notice(user, old_key).deliver_later
      
      # Log rotation
      Rails.logger.info "Rotated API key for user #{user.id}"
    end
  end
end

# Run periodically
# RotateApiKeys.new.call
```

## Best Practices Summary

1. **Start with simple API keys**
2. **Use HTTPS always**
3. **Implement rate limiting**
4. **Log authentication attempts**
5. **Rotate tokens regularly**
6. **Use constant-time comparisons**
7. **Never log sensitive tokens**
8. **Test authentication thoroughly**