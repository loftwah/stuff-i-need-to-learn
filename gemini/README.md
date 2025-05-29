# Google Gemini AI Integration Guide for Rails

Complete guide for integrating Google Gemini 2.5 Flash with Rails 8, including setup, usage patterns, and console workflows.

## Gemini 2.5 Flash Overview

### Model Specifications

| Feature           | Value                            |
| ----------------- | -------------------------------- |
| Model ID          | `gemini-2.5-flash-preview-05-20` |
| Launch Stage      | Public Preview (May 20, 2025)    |
| Max Input Tokens  | 1,048,576 (~1M tokens)           |
| Max Output Tokens | 65,535                           |
| Knowledge Cutoff  | January 2025                     |

### Supported I/O

| Type   | Input | Output | Notes                                      |
| ------ | ----- | ------ | ------------------------------------------ |
| Text   | ✅    | ✅     | Primary use case                           |
| Code   | ✅    | ✅     | Programming assistance                     |
| Images | ✅    | ❌     | Up to 3,000 images, 7MB each               |
| Audio  | ✅    | ❌     | Up to 8.4 hours, transcription/translation |
| Video  | ✅    | ❌     | Up to 45min with audio, 60min without      |
| JSON   | ✅    | ✅     | Structured input/output                    |

### Key Capabilities

- **Thinking Preview** - Shows reasoning steps
- **Function Calling** - Tool integration with JSON schemas
- **System Instructions** - Role/personality preloading
- **Controlled Generation** - JSON output with schemas
- **Context Caching** - Conversation persistence
- **Grounding** - Google Search integration

## Google Cloud Setup

### 1. Create Project & Enable APIs

```bash
# Go to: https://console.cloud.google.com/projectcreate
# Project name: gemini-rails-app

# Enable required APIs:
# - Vertex AI API: https://console.cloud.google.com/apis/library/aiplatform.googleapis.com
# - Enable billing for your project
```

### 2. Create Service Account

```bash
# Go to: https://console.cloud.google.com/iam-admin/serviceaccounts
# Create service account: gemini-service

# Assign roles:
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:gemini-service@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user"

gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:gemini-service@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/ml.admin"
```

### 3. Download Credentials

```bash
# Download JSON key file as google-credentials.json
# Convert to single-line for .env:
jq -c '.' google-credentials.json
```

## Rails 8 Integration

### 1. Add Gem

```ruby
# Gemfile
gem 'gemini-ai', '~> 4.2.0'
```

### 2. Environment Configuration

```bash
# .env (development/test)
GOOGLE_CREDENTIALS_FILE_CONTENTS='{"type":"service_account","project_id":"your-project",...}'
GOOGLE_GEMINI_REGION=us-central1
GOOGLE_PROJECT_ID=your-project-id
```

```yaml
# config/credentials.yml.enc (production)
# EDITOR="code --wait" bin/rails credentials:edit
google:
  gemini:
    credentials_file_contents: '{"type":"service_account",...}'
    region: us-central1
    project_id: your-project-id
```

### 3. Client Initializer

```ruby
# config/initializers/gemini.rb
require 'gemini-ai'

credentials = if Rails.env.production?
  {
    service: 'vertex-ai-api',
    file_contents: Rails.application.credentials.dig(:google, :gemini, :credentials_file_contents),
    region: Rails.application.credentials.dig(:google, :gemini, :region),
    project_id: Rails.application.credentials.dig(:google, :gemini, :project_id)
  }
else
  {
    service: 'vertex-ai-api',
    file_contents: ENV['GOOGLE_CREDENTIALS_FILE_CONTENTS'],
    region: ENV.fetch('GOOGLE_GEMINI_REGION', 'us-central1'),
    project_id: ENV['GOOGLE_PROJECT_ID']
  }
end

GEMINI_CLIENT = Gemini.new(
  credentials: credentials,
  options: {
    model: 'gemini-2.5-flash-preview-05-20',
    server_sent_events: true
  }
)
```

## Basic Usage Patterns

### Simple Text Generation

```ruby
result = GEMINI_CLIENT.generate_content(
  contents: { role: 'user', parts: { text: 'Explain Ruby on Rails' } }
)

response_text = result.dig('candidates', 0, 'content', 'parts', 0, 'text')
puts response_text
```

### Streaming Response

```ruby
GEMINI_CLIENT.stream_generate_content(
  { contents: { role: 'user', parts: { text: 'Write a story about AI' } } }
) do |event, parsed, raw|
  text = parsed.dig('candidates', 0, 'content', 'parts', 0, 'text')
  print text if text
end
```

### System Instructions

```ruby
result = GEMINI_CLIENT.generate_content({
  contents: { role: 'user', parts: { text: 'Who are you?' } },
  system_instruction: {
    role: 'user',
    parts: { text: 'You are a helpful Ruby on Rails expert named Claude.' }
  }
})
```

## JSON Format Responses

_As of the writing of this README, only the `vertex-ai-api` service and `gemini` models version `1.5` support this feature._

The Gemini API provides a configuration parameter to request a response in JSON format:

```ruby
require 'json'

result = GEMINI_CLIENT.stream_generate_content({
  contents: {
    role: 'user',
    parts: {
      text: 'List 3 random colors.'
    }
  },
  generation_config: {
    response_mime_type: 'application/json'
  }
})

json_string = result
              .map { |response| response.dig('candidates', 0, 'content', 'parts') }
              .map { |parts| parts.map { |part| part['text'] }.join }
              .join

puts JSON.parse(json_string).inspect
```

Output:

```ruby
{ 'colors' => ['Dark Salmon', 'Indigo', 'Lavender'] }
```

## JSON Schema

_While Gemini 1.5 Flash models only accept a text description of the JSON schema you want returned, the Gemini 1.5 Pro models let you pass a schema object (or a Python type equivalent), and the model output will strictly follow that schema. This is also known as controlled generation or constrained decoding._

You can also provide a JSON Schema for the expected JSON output:

```ruby
require 'json'

result = GEMINI_CLIENT.stream_generate_content({
  contents: {
    role: 'user',
    parts: {
      text: 'List 3 random colors.'
    }
  },
  generation_config: {
    response_mime_type: 'application/json',
    response_schema: {
      type: 'object',
      properties: {
        colors: {
          type: 'array',
          items: {
            type: 'object',
            properties: {
              name: {
                type: 'string'
              }
            }
          }
        }
      }
    }
  }
})

json_string = result
              .map { |response| response.dig('candidates', 0, 'content', 'parts') }
              .map { |parts| parts.map { |part| part['text'] }.join }
              .join

puts JSON.parse(json_string).inspect
```

Output:

```ruby
{ 'colors' => [
  { 'name' => 'Lavender Blush' },
  { 'name' => 'Medium Turquoise' },
  { 'name' => 'Dark Slate Gray' }
] }
```

## TechDeck Examples

### User Card Generation with System Instructions + JSON Schema

```ruby
# Build system instruction
system_instruction = {
  role: 'user',
  parts: {
    text: <<~INSTRUCTION
      You are an expert at analyzing social media profiles and creating engaging character profiles.
      You must follow character limits EXACTLY or your output will be rejected.
      Always respond in the specified JSON format with no additional text.
    INSTRUCTION
  }
}

# Define JSON schema for user card generation
user_card_schema = {
  type: 'object',
  properties: {
    short_bio: {
      type: 'string',
      description: 'Third-person bio, 150-250 characters exactly'
    },
    long_bio: {
      type: 'string',
      description: 'Detailed third-person bio, 900-1100 characters with newlines'
    },
    tags: {
      type: 'array',
      items: { type: 'string' },
      minItems: 6,
      maxItems: 6,
      description: 'Exactly 6 lowercase single words'
    },
    buff: {
      type: 'string',
      maxLength: 30,
      description: 'Title Case video game attribute'
    },
    weakness: {
      type: 'string',
      maxLength: 30,
      description: 'Title Case video game weakness'
    },
    vibe: {
      type: 'string',
      maxLength: 30,
      description: 'Title Case creative descriptor'
    },
    special_move: {
      type: 'string',
      maxLength: 30,
      description: 'Title Case special ability name'
    },
    flavor_text: {
      type: 'string',
      minLength: 30,
      maxLength: 60,
      description: 'Creative quote or mantra'
    },
    buff_description: {
      type: 'string',
      maxLength: 100,
      description: 'Explains what the buff does'
    },
    weakness_description: {
      type: 'string',
      maxLength: 100,
      description: 'Explains the weakness'
    },
    vibe_description: {
      type: 'string',
      maxLength: 100,
      description: 'Elaborates on the vibe'
    },
    special_move_description: {
      type: 'string',
      maxLength: 100,
      description: 'Describes the special move'
    }
  },
  required: [
    'short_bio', 'long_bio', 'tags', 'buff', 'weakness',
    'vibe', 'special_move', 'flavor_text', 'buff_description',
    'weakness_description', 'vibe_description', 'special_move_description'
  ]
}

# Make request with system instruction and schema
def generate_user_card(user, tweets)
  tweet_sample = tweets.limit(10).pluck(:text).join(' | ')

  prompt = <<~PROMPT
    Create a tech trading card profile for #{user.name} (@#{user.handle}).

    PROFILE DATA:
    - Bio: #{user.bio}
    - Location: #{user.location}
    - Followers: #{user.followers}
    - Following: #{user.following}
    - Verified: #{user.verified?}
    - Recent content: #{tweet_sample.truncate(600)}

    CRITICAL REQUIREMENTS:
    - Character limits are STRICT
    - Third-person perspective for bios
    - Tags must be single words, lowercase
    - Content should be professional yet engaging
    - Reflect their actual expertise and personality
  PROMPT

  result = GEMINI_CLIENT.stream_generate_content({
    system_instruction: system_instruction,
    contents: {
      role: 'user',
      parts: { text: prompt }
    },
    generation_config: {
      response_mime_type: 'application/json',
      response_schema: user_card_schema
    }
  })

  json_string = result
                .map { |response| response.dig('candidates', 0, 'content', 'parts') }
                .map { |parts| parts.map { |part| part['text'] }.join }
                .join

  JSON.parse(json_string)
end
```

### Schema Validation Helper

```ruby
# Helper to validate response against schema
def validate_json_response(json_string, schema)
  data = JSON.parse(json_string)

  # Check required fields
  schema[:required]&.each do |field|
    unless data.key?(field.to_s)
      raise "Missing required field: #{field}"
    end
  end

  # Check string lengths
  schema[:properties].each do |field, rules|
    value = data[field.to_s]
    next unless value.is_a?(String)

    if rules[:maxLength] && value.length > rules[:maxLength]
      raise "#{field} too long: #{value.length} > #{rules[:maxLength]}"
    end

    if rules[:minLength] && value.length < rules[:minLength]
      raise "#{field} too short: #{value.length} < #{rules[:minLength]}"
    end
  end

  data
end
```

## Rails Console Workflows

### Working with TechDeck Models

```ruby
# Start Rails console
bin/rails console

# Get a user with tweets
user = User.includes(:tweets).find_by(handle: 'loftwah')
tweets = user.tweets.limit(10)

# Simple profile analysis
prompt = <<~PROMPT
  Analyze this Twitter user and suggest improvements:

  Name: #{user.name}
  Handle: @#{user.handle}
  Bio: #{user.bio}
  Followers: #{user.followers}
  Following: #{user.following}
  Recent tweets: #{tweets.pluck(:text).join('; ')}
PROMPT

result = GEMINI_CLIENT.generate_content(
  contents: { role: 'user', parts: { text: prompt } }
)

puts result.dig('candidates', 0, 'content', 'parts', 0, 'text')
```

### Generate Card Stats with AI Input

```ruby
# Get AI suggestions for card attributes
user = User.find_by(handle: 'metaprinxss')
tweets = user.tweets.order(created_at_twitter: :desc).limit(20)

ai_prompt = <<~PROMPT
  Based on this user's profile, suggest RPG-style attributes:

  #{user.attributes.slice('name', 'handle', 'bio', 'followers', 'following').to_json}

  Recent activity: #{tweets.pluck(:text).join('; ')}

  Suggest appropriate:
  1. Playing card (rank and suit)
  2. Australian spirit animal
  3. RPG stats explanation (why they'd have high/low attack/defense/speed)
PROMPT

ai_result = GEMINI_CLIENT.generate_content(
  contents: { role: 'user', parts: { text: ai_prompt } }
)

puts ai_result.dig('candidates', 0, 'content', 'parts', 0, 'text')

# Generate actual stats
stats_service = CardStatService.new(user, tweets: tweets)
stats = stats_service.generate_stats
pp stats
```

### Test Your AiGenerationService

```ruby
# Test the existing service
user = User.find_by(handle: 'loftwah')
ai_service = AiGenerationService.new(user)

# Check what prompt it builds
prompt = ai_service.send(:build_prompt)
puts prompt

# Test the generation (this will actually update the user)
success = ai_service.generate!
puts "Generation #{success ? 'succeeded' : 'failed'}"

# Check what was generated
user.reload
puts "Short bio: #{user.short_bio}"
puts "Long bio: #{user.long_bio}"
puts "Tags: #{user.tags.pluck(:name)}"
puts "Buff: #{user.buff} - #{user.buff_description}"
```

### Custom JSON Schema Testing

```ruby
# Define a simple schema for tweet analysis
tweet_analysis_schema = {
  type: 'object',
  properties: {
    sentiment: {
      type: 'string',
      enum: ['positive', 'negative', 'neutral']
    },
    topics: {
      type: 'array',
      items: { type: 'string' },
      maxItems: 5
    },
    engagement_prediction: {
      type: 'integer',
      minimum: 1,
      maximum: 100
    },
    audience_type: {
      type: 'string',
      enum: ['technical', 'general', 'professional', 'casual']
    }
  },
  required: ['sentiment', 'topics', 'engagement_prediction', 'audience_type']
}

# Test with a user's tweets
user = User.find_by(handle: 'loftwah')
recent_tweets = user.tweets.limit(5).pluck(:text)

result = GEMINI_CLIENT.stream_generate_content({
  contents: {
    role: 'user',
    parts: {
      text: "Analyze these tweets: #{recent_tweets.join(' | ')}"
    }
  },
  generation_config: {
    response_mime_type: 'application/json',
    response_schema: tweet_analysis_schema
  }
})

json_string = result
              .map { |response| response.dig('candidates', 0, 'content', 'parts') }
              .map { |parts| parts.map { |part| part['text'] }.join }
              .join

analysis = JSON.parse(json_string)
pp analysis
```

### Batch User Processing

```ruby
# Process multiple users with different prompts
usernames = ['loftwah', 'metaprinxss']
results = {}

usernames.each do |username|
  user = User.find_by(handle: username)
  next unless user

  # Custom analysis for each user
  system_instruction = {
    role: 'user',
    parts: {
      text: "You are analyzing a TechDeck trading card user. Focus on their technical expertise and online presence."
    }
  }

  result = GEMINI_CLIENT.generate_content({
    system_instruction: system_instruction,
    contents: {
      role: 'user',
      parts: {
        text: "Analyze @#{username}: #{user.bio} | Followers: #{user.followers}"
      }
    }
  })

  results[username] = result.dig('candidates', 0, 'content', 'parts', 0, 'text')

  # Rate limiting
  sleep(1)
end

pp results
```

### Data Format Conversions

```ruby
# Get user data in different formats
user = User.includes(:tweets, :tags).first

# JSON format (most common for Gemini)
user_json = {
  profile: user.attributes.slice('name', 'handle', 'bio', 'followers', 'following'),
  recent_tweets: user.tweets.limit(10).pluck(:text),
  tags: user.tags.pluck(:name),
  stats: {
    attack: user.attack,
    defense: user.defense,
    speed: user.speed
  }
}.to_json

puts "JSON for Gemini:"
puts JSON.pretty_generate(JSON.parse(user_json))

# YAML format (good for configuration-style prompts)
user_yaml = {
  profile: user.attributes.slice('name', 'handle', 'bio'),
  metrics: {
    social: { followers: user.followers, following: user.following },
    content: { tweets: user.tweets.count, tags: user.tags.count }
  }
}.to_yaml

puts "\nYAML format:"
puts user_yaml

# CSV-style for tabular analysis
require 'csv'
csv_data = CSV.generate do |csv|
  csv << ['Handle', 'Name', 'Followers', 'Following', 'Tweets', 'Bio']
  User.limit(5).each do |u|
    csv << [u.handle, u.name, u.followers, u.following, u.tweets.count, u.bio&.truncate(50)]
  end
end

puts "\nCSV format:"
puts csv_data
```

### System Instructions with Context

```ruby
# Create context-aware system instructions
def create_techdeck_system_instruction(context_type = :general)
  instructions = {
    general: "You are analyzing TechDeck trading card profiles. Focus on technical skills, online presence, and creating engaging character descriptions.",

    stats_generation: <<~INSTRUCTION,
      You are a game designer creating RPG-style character stats for tech professionals.
      Consider their followers (influence), tweet frequency (activity), and bio content (expertise).
      Assign stats that reflect their real-world impact in the tech community.
    INSTRUCTION

    content_creation: <<~INSTRUCTION,
      You are a creative writer specializing in character profiles for tech trading cards.
      Create engaging, accurate descriptions that highlight the person's technical expertise
      and personality. Use gaming terminology and Australian cultural references when appropriate.
    INSTRUCTION
  }

  {
    role: 'user',
    parts: { text: instructions[context_type] }
  }
end

# Use different system instructions
user = User.find_by(handle: 'loftwah')

# For stats analysis
stats_analysis = GEMINI_CLIENT.generate_content({
  system_instruction: create_techdeck_system_instruction(:stats_generation),
  contents: {
    role: 'user',
    parts: {
      text: "Explain why #{user.name} (@#{user.handle}) would have these stats: Attack=#{user.attack}, Defense=#{user.defense}, Speed=#{user.speed}"
    }
  }
})

puts "Stats explanation:"
puts stats_analysis.dig('candidates', 0, 'content', 'parts', 0, 'text')

# For content creation
content_result = GEMINI_CLIENT.generate_content({
  system_instruction: create_techdeck_system_instruction(:content_creation),
  contents: {
    role: 'user',
    parts: {
      text: "Create a trading card flavor text for #{user.name}: #{user.bio}"
    }
  }
})

puts "\nFlavor text:"
puts content_result.dig('candidates', 0, 'content', 'parts', 0, 'text')
```

## Image Analysis

```ruby
require 'base64'

# Load image from Rails assets
image_path = Rails.root.join('app/assets/images/sample.jpg')
image_data = Base64.strict_encode64(File.read(image_path))

result = GEMINI_CLIENT.stream_generate_content({
  contents: [{
    role: 'user',
    parts: [
      { text: 'What do you see in this image?' },
      {
        inline_data: {
          mime_type: 'image/jpeg',
          data: image_data
        }
      }
    ]
  }]
})

json_string = result
              .map { |response| response.dig('candidates', 0, 'content', 'parts') }
              .map { |parts| parts.map { |part| part['text'] }.join }
              .join

puts json_string
```

### MIME Type Detection

```ruby
require 'marcel'

# Detect MIME type for file uploads
file_path = 'path/to/file'
mime_type = Marcel::MimeType.for(Pathname.new(file_path))

# Verify supported types
supported_image_types = %w[image/png image/jpeg image/webp]
supported_audio_types = %w[audio/mp3 audio/wav audio/flac audio/m4a]
supported_video_types = %w[video/mp4 video/webm video/quicktime]

if supported_image_types.include?(mime_type)
  # Process as image
elsif supported_audio_types.include?(mime_type)
  # Process as audio
end
```

## Conversation Management

### Back-and-Forth Chat

```ruby
conversation = [
  { role: 'user', parts: { text: 'Hi, my name is Dean' } }
]

# First response
result = GEMINI_CLIENT.generate_content(contents: conversation)
model_response = result.dig('candidates', 0, 'content')

# Add to conversation history
conversation << model_response
conversation << { role: 'user', parts: { text: "What's my name?" } }

# Continue conversation
result = GEMINI_CLIENT.generate_content(contents: conversation)
puts result.dig('candidates', 0, 'content', 'parts', 0, 'text')
```

### Session Management Service

```ruby
# app/services/gemini_chat_service.rb
class GeminiChatService
  def initialize(session_id = nil)
    @session_id = session_id || SecureRandom.uuid
    @conversation = load_conversation
  end

  def chat(message, system_instruction: nil)
    # Add user message
    @conversation << { role: 'user', parts: { text: message } }

    # Build request
    request = { contents: @conversation }
    if system_instruction
      request[:system_instruction] = {
        role: 'user',
        parts: { text: system_instruction }
      }
    end

    # Get response
    result = GEMINI_CLIENT.generate_content(request)
    model_response = result.dig('candidates', 0, 'content')

    # Add to conversation
    @conversation << model_response
    save_conversation

    model_response.dig('parts', 0, 'text')
  end

  private

  def load_conversation
    Rails.cache.read("gemini_chat_#{@session_id}") || []
  end

  def save_conversation
    Rails.cache.write("gemini_chat_#{@session_id}", @conversation, expires_in: 1.hour)
  end
end
```

## Function Calling (Tools)

### Define Functions

```ruby
tools = {
  function_declarations: [
    {
      name: 'get_user_count',
      description: 'Returns the total number of users in the database',
      parameters: {
        type: 'object',
        properties: {
          active_only: {
            type: 'boolean',
            description: 'If true, only count active users'
          }
        }
      }
    },
    {
      name: 'create_user_summary',
      description: 'Creates a summary of user data',
      parameters: {
        type: 'object',
        properties: {
          user_id: {
            type: 'integer',
            description: 'The ID of the user to summarize'
          }
        }
      }
    }
  ]
}

# Make request with tools
result = GEMINI_CLIENT.stream_generate_content({
  tools: tools,
  contents: [
    { role: 'user', parts: { text: 'How many active users do we have?' } }
  ]
})

# Check for function calls
result.each do |response|
  parts = response.dig('candidates', 0, 'content', 'parts') || []

  parts.each do |part|
    if part.key?('functionCall')
      function_name = part.dig('functionCall', 'name')
      args = part.dig('functionCall', 'args')

      case function_name
      when 'get_user_count'
        count = args['active_only'] ? User.active.count : User.count
        # Return result back to Gemini...
      end
    end
  end
end
```

## Error Handling

### Comprehensive Error Handling

```ruby
require 'gemini-ai/errors'

def safe_gemini_request(prompt, max_retries: 3)
  retries = 0

  begin
    GEMINI_CLIENT.generate_content(
      contents: { role: 'user', parts: { text: prompt } }
    )
  rescue => e
    retries += 1

    if retries <= max_retries
      sleep(2 ** retries) # Exponential backoff
      retry
    else
      Rails.logger.error "Gemini API Error: #{e.message}"
      raise
    end
  end
end

# Usage
begin
  result = safe_gemini_request("Explain machine learning")
  puts result.dig('candidates', 0, 'content', 'parts', 0, 'text')
rescue => e
  puts "Failed to get response: #{e.message}"
end
```

## Performance & Optimization

### Token Counting

```ruby
# Count tokens before sending
token_count = GEMINI_CLIENT.count_tokens(
  contents: { role: 'user', parts: { text: 'Your prompt here' } }
)
puts "This request will use #{token_count['totalTokens']} tokens"
```

### Context Caching

```ruby
# For repeated similar requests, cache context
Rails.cache.fetch("gemini_analysis_#{data.hash}", expires_in: 1.hour) do
  GEMINI_CLIENT.generate_content(
    contents: { role: 'user', parts: { text: "Analyze: #{data.to_json}" } }
  )
end
```

### Batch Processing

```ruby
def process_in_batches(items, batch_size: 5)
  items.each_slice(batch_size) do |batch|
    batch.each do |item|
      result = GEMINI_CLIENT.generate_content(
        contents: { role: 'user', parts: { text: "Process: #{item}" } }
      )
      # Process result...
    end
    sleep(1) # Rate limiting
  end
end
```

## Testing Patterns

### RSpec Setup

```ruby
# spec/rails_helper.rb
require 'webmock/rspec'

RSpec.configure do |config|
  config.before(:each) do
    stub_request(:post, /aiplatform\.googleapis\.com/)
      .to_return(
        status: 200,
        body: {
          candidates: [{
            content: {
              parts: [{ text: 'Mocked response' }]
            }
          }]
        }.to_json,
        headers: { 'Content-Type' => 'application/json' }
      )
  end
end
```

### Service Testing

```ruby
# spec/services/gemini_service_spec.rb
RSpec.describe GeminiService do
  describe '#analyze_text' do
    it 'returns analysis for given text' do
      service = GeminiService.new
      result = service.analyze_text('Test content')

      expect(result).to be_a(String)
      expect(result).not_to be_empty
    end
  end
end
```

## Supported Regions

| Region          | Location          | Notes                                          |
| --------------- | ----------------- | ---------------------------------------------- |
| us-central1     | Iowa              | **Required for preview models like 2.5 Flash** |
| us-east4        | Northern Virginia | Standard models only                           |
| us-west1        | Oregon            | Standard models only                           |
| us-west4        | Las Vegas         | Standard models only                           |
| asia-northeast1 | Tokyo             | Standard models only                           |
| asia-southeast1 | Singapore         | Standard models only                           |

**Important:** Preview models (including `gemini-2.5-flash-preview-05-20`) are only available in `us-central1`.

## Common Issues & Solutions

### Issue: "Model not found"

**Solution:** Ensure you're using `us-central1` region for preview models like `gemini-2.5-flash-preview-05-20`

### Issue: "Permission denied"

**Solution:** Verify IAM roles (`roles/aiplatform.user` and `roles/ml.admin`)

### Issue: "Invalid credentials"

**Solution:** Check JSON format in environment variables (use `jq -c` for single-line)

### Issue: "Quota exceeded"

**Solution:** Implement exponential backoff and rate limiting

### Issue: JSON Schema not enforced

**Solution:** Only `gemini-1.5-pro` and newer models support strict schema enforcement. Flash models work best with detailed text descriptions in the schema.

### Issue: Character limits not respected

**Solution:** Use system instructions to emphasize limits, validate responses, and retry if needed:

```ruby
def validate_and_retry(user, max_attempts: 3)
  attempts = 0

  loop do
    attempts += 1
    result = AiGenerationService.new(user).generate!

    # Validate character limits
    if user.short_bio&.length&.between?(150, 250) &&
       user.long_bio&.length&.between?(900, 1100)
      return true
    end

    break if attempts >= max_attempts
    Rails.logger.warn "Attempt #{attempts} failed validation, retrying..."
  end

  false
end
```

This comprehensive guide provides everything you need to integrate Gemini AI into your Rails application with proper error handling, optimization, and real-world usage patterns.
