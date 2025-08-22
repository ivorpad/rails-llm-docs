# Error Handling Best Practices

## Core Principle
Handle errors gracefully at the appropriate level. Use result objects for expected failures, exceptions for unexpected failures, and always provide useful context.

## Result Objects for Expected Failures

### Basic Result Pattern
```ruby
class Result
  attr_reader :value, :error
  
  def self.success(value = nil)
    new(success: true, value: value)
  end
  
  def self.failure(error)
    new(success: false, error: error)
  end
  
  def initialize(success:, value: nil, error: nil)
    @success = success
    @value = value
    @error = error
  end
  
  def success?
    @success
  end
  
  def failure?
    !@success
  end
  
  def on_success
    yield(value) if success? && block_given?
    self
  end
  
  def on_failure
    yield(error) if failure? && block_given?
    self
  end
end
```

### Using Result Objects in Services
```ruby
class CreateUser
  def call(email:, password:, name:)
    # Validation
    validation_result = validate_inputs(email, password)
    return validation_result if validation_result.failure?
    
    # Check for existing user
    if User.exists?(email: email)
      return Result.failure(
        code: :email_taken,
        message: "Email is already registered"
      )
    end
    
    # Create user
    user = User.new(email: email, password: password, name: name)
    
    if user.save
      UserMailer.welcome(user).deliver_later
      Result.success(user)
    else
      Result.failure(
        code: :validation_failed,
        message: "Could not create user",
        errors: user.errors.full_messages
      )
    end
  rescue ActiveRecord::RecordNotUnique
    Result.failure(
      code: :race_condition,
      message: "Email was taken during registration"
    )
  end
  
  private
  
  def validate_inputs(email, password)
    return Result.failure(code: :invalid_email, message: "Email is invalid") unless email.match?(URI::MailTo::EMAIL_REGEXP)
    return Result.failure(code: :weak_password, message: "Password must be at least 8 characters") if password.length < 8
    
    Result.success
  end
end

# Controller usage
class UsersController < ApplicationController
  def create
    result = CreateUser.new.call(
      email: params[:email],
      password: params[:password],
      name: params[:name]
    )
    
    result
      .on_success { |user| redirect_to user, notice: "Welcome!" }
      .on_failure { |error| handle_error(error) }
  end
  
  private
  
  def handle_error(error)
    case error[:code]
    when :email_taken
      @user = User.new(user_params)
      flash.now[:alert] = error[:message]
      render :new
    when :validation_failed
      @user = User.new(user_params)
      @errors = error[:errors]
      render :new
    else
      redirect_to signup_path, alert: "An error occurred. Please try again."
    end
  end
end
```

## Exception Handling Strategies

### Application-Wide Error Handler
```ruby
class ApplicationController < ActionController::Base
  rescue_from StandardError, with: :handle_error
  rescue_from ActiveRecord::RecordNotFound, with: :not_found
  rescue_from ActionController::ParameterMissing, with: :bad_request
  rescue_from Pundit::NotAuthorizedError, with: :forbidden
  
  private
  
  def handle_error(exception)
    # Log the error with context
    Rails.logger.error({
      error: exception.class.name,
      message: exception.message,
      backtrace: exception.backtrace[0..10],
      user_id: current_user&.id,
      request_id: request.request_id,
      params: params.to_unsafe_h
    }.to_json)
    
    # Notify error tracking service
    Sentry.capture_exception(exception)
    
    # Show user-friendly error
    respond_to do |format|
      format.html { 
        render file: 'public/500.html', 
               status: :internal_server_error, 
               layout: false 
      }
      format.json { 
        render json: { error: 'Internal server error' }, 
               status: :internal_server_error 
      }
    end
  end
  
  def not_found
    respond_to do |format|
      format.html { render file: 'public/404.html', status: :not_found, layout: false }
      format.json { render json: { error: 'Not found' }, status: :not_found }
    end
  end
  
  def bad_request(exception)
    respond_to do |format|
      format.html { redirect_to root_path, alert: 'Invalid request' }
      format.json { 
        render json: { error: exception.message }, 
               status: :bad_request 
      }
    end
  end
  
  def forbidden
    respond_to do |format|
      format.html { redirect_to root_path, alert: 'Not authorized' }
      format.json { render json: { error: 'Forbidden' }, status: :forbidden }
    end
  end
end
```

## Custom Error Classes

### Domain-Specific Errors
```ruby
module Errors
  class ApplicationError < StandardError
    attr_reader :code, :status, :details
    
    def initialize(message = nil, code: nil, status: nil, details: {})
      super(message)
      @code = code
      @status = status
      @details = details
    end
  end
  
  class ValidationError < ApplicationError
    def initialize(message = "Validation failed", details: {})
      super(message, code: :validation_error, status: 422, details: details)
    end
  end
  
  class AuthenticationError < ApplicationError
    def initialize(message = "Authentication required")
      super(message, code: :unauthenticated, status: 401)
    end
  end
  
  class AuthorizationError < ApplicationError
    def initialize(message = "Not authorized", resource: nil)
      super(message, code: :unauthorized, status: 403, details: { resource: resource })
    end
  end
  
  class PaymentError < ApplicationError
    def initialize(message, gateway_error: nil)
      super(message, code: :payment_failed, status: 402, details: { gateway_error: gateway_error })
    end
  end
  
  class ExternalServiceError < ApplicationError
    def initialize(service, message)
      super("#{service} error: #{message}", code: :external_service_error, status: 503)
    end
  end
end

# Usage
class PaymentService
  def charge(amount:, token:)
    response = PaymentGateway.charge(amount, token)
    
    unless response.success?
      raise Errors::PaymentError.new(
        "Payment failed: #{response.error_message}",
        gateway_error: response.error_code
      )
    end
    
    response
  end
end
```

## Error Context and Logging

### Structured Error Logging
```ruby
class ErrorLogger
  def self.log(exception, context = {})
    error_data = {
      error_class: exception.class.name,
      error_message: exception.message,
      error_backtrace: exception.backtrace&.first(5),
      timestamp: Time.current.iso8601,
      environment: Rails.env,
      **context
    }
    
    Rails.logger.error(error_data.to_json)
    
    # Send to monitoring service
    if Rails.env.production?
      Sentry.capture_exception(exception, extra: context)
    end
  end
end

# Usage in service
class ProcessOrder
  def call(order_id:)
    order = Order.find(order_id)
    
    # Process order...
    
  rescue => e
    ErrorLogger.log(e, {
      service: self.class.name,
      order_id: order_id,
      user_id: order&.user_id,
      order_total: order&.total
    })
    
    raise
  end
end
```

## Graceful Degradation

### Circuit Breaker Pattern
```ruby
class CircuitBreaker
  attr_reader :name, :threshold, :timeout
  
  def initialize(name:, threshold: 5, timeout: 60)
    @name = name
    @threshold = threshold
    @timeout = timeout
  end
  
  def call
    if open?
      raise Errors::ExternalServiceError.new(name, "Circuit breaker is open")
    end
    
    begin
      result = yield
      reset_failures
      result
    rescue => e
      record_failure
      raise
    end
  end
  
  private
  
  def open?
    failure_count >= threshold && !timeout_expired?
  end
  
  def failure_count
    Redis.current.get("circuit_breaker:#{name}:failures").to_i
  end
  
  def record_failure
    Redis.current.multi do |r|
      r.incr("circuit_breaker:#{name}:failures")
      r.expire("circuit_breaker:#{name}:failures", timeout)
    end
  end
  
  def reset_failures
    Redis.current.del("circuit_breaker:#{name}:failures")
  end
  
  def timeout_expired?
    !Redis.current.exists?("circuit_breaker:#{name}:failures")
  end
end

# Usage
class ExternalApiService
  def fetch_data
    circuit_breaker.call do
      response = HTTParty.get('https://api.example.com/data')
      
      raise "API error" unless response.success?
      
      response.parsed_response
    end
  rescue Errors::ExternalServiceError
    # Return cached or default data
    Rails.cache.fetch('external_api_data', expires_in: 1.hour) do
      { default: 'data' }
    end
  end
  
  private
  
  def circuit_breaker
    @circuit_breaker ||= CircuitBreaker.new(
      name: 'external_api',
      threshold: 3,
      timeout: 30
    )
  end
end
```

## API Error Responses

### Consistent API Errors
```ruby
module Api
  class ErrorRenderer
    def self.render(error)
      case error
      when ActiveRecord::RecordNotFound
        {
          error: {
            type: 'not_found',
            message: 'Resource not found'
          }
        }
      when ActiveRecord::RecordInvalid
        {
          error: {
            type: 'validation_error',
            message: 'Validation failed',
            details: error.record.errors.as_json
          }
        }
      when Errors::ApplicationError
        {
          error: {
            type: error.code,
            message: error.message,
            details: error.details
          }
        }
      else
        {
          error: {
            type: 'internal_error',
            message: 'An error occurred'
          }
        }
      end
    end
  end
end

class Api::BaseController < ActionController::API
  rescue_from StandardError do |exception|
    error_response = Api::ErrorRenderer.render(exception)
    status = exception.respond_to?(:status) ? exception.status : 500
    
    render json: error_response, status: status
  end
end
```

## Form Error Handling

### User-Friendly Form Errors
```ruby
class ApplicationForm
  include ActiveModel::Model
  
  def save
    return false unless valid?
    
    persist!
    true
  rescue => e
    handle_persistence_error(e)
    false
  end
  
  private
  
  def handle_persistence_error(error)
    case error
    when ActiveRecord::RecordNotUnique
      errors.add(:base, "This record already exists")
    when ActiveRecord::RecordInvalid
      error.record.errors.each do |error|
        errors.add(error.attribute, error.message)
      end
    else
      errors.add(:base, "An error occurred. Please try again.")
      ErrorLogger.log(error, form: self.class.name)
    end
  end
end

class RegistrationForm < ApplicationForm
  attr_accessor :email, :password, :name
  
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, length: { minimum: 8 }
  validates :name, presence: true
  
  private
  
  def persist!
    User.create!(email: email, password: password, name: name)
  end
end
```

## Testing Error Handling

### RSpec Examples
```ruby
RSpec.describe CreateUser do
  describe "#call" do
    context "when email is taken" do
      before { create(:user, email: "taken@example.com") }
      
      it "returns failure result" do
        result = described_class.new.call(
          email: "taken@example.com",
          password: "password123",
          name: "John"
        )
        
        expect(result).to be_failure
        expect(result.error[:code]).to eq(:email_taken)
      end
    end
    
    context "when external service fails" do
      it "handles the error gracefully" do
        allow(UserMailer).to receive(:welcome).and_raise(Net::SMTPError)
        
        expect(ErrorLogger).to receive(:log)
        
        result = described_class.new.call(
          email: "new@example.com",
          password: "password123",
          name: "John"
        )
        
        # User is still created even if email fails
        expect(result).to be_success
        expect(User.find_by(email: "new@example.com")).to be_present
      end
    end
  end
end

RSpec.describe "Error pages", type: :request do
  it "shows 404 page for missing resources" do
    get "/products/999999"
    
    expect(response).to have_http_status(:not_found)
    expect(response.body).to include("Page not found")
  end
  
  it "shows 500 page for server errors" do
    allow_any_instance_of(ProductsController).to receive(:index).and_raise(StandardError)
    
    get "/products"
    
    expect(response).to have_http_status(:internal_server_error)
    expect(response.body).to include("We're sorry")
  end
end
```

## Best Practices Summary

1. **Use result objects for expected failures**
2. **Raise exceptions for unexpected failures**
3. **Provide useful error context**
4. **Log errors with structured data**
5. **Handle errors at appropriate levels**
6. **Implement graceful degradation**
7. **Keep error messages user-friendly**
8. **Test error paths thoroughly**