# How to setup an app with GraphQL, Rails, React, Relay

## Generate a new app

1. rails new APP_NAME --webpack=react --database=postgresql --skip-test --skip-coffee
  - --skip-test, --database, and --skip-coffee is optional
1. add `react-rails` to Gemfile
1. add `graphql` to Gemfile
1. run `bundle install`
1. run `rails g graphql:install`
1. run `rails g react:install`
1. run `yarn add react-relay@^1.0.0`
1. run `yarn add relay-compiler@^1.0.0 babel-plugin-relay@^1.0.1 --dev`
1. add `<%= javascript_pack_tag 'application' %>` to `app/views/layouts/application.html.erb`
1. add `<%= stylesheet_pack_tag 'application' %>` to `app/views/layouts/application.html.erb`
1. move `app/javascript/packs/hello_react.jsx` to `app/javascript/components/hello_react.jsx`
1. modify the generated GraphQL schema file which should be under `app/graphql/app_name_schema.rb`

```ruby
AppNameSchema = GraphQL::Schema.define do
  query(Types::QueryType)
end

module RelaySchemaHelpers
  # Schema.json location
  SCHEMA_DIR  = Rails.root.join('app/javascript/packs/')
  SCHEMA_PATH = File.join(SCHEMA_DIR, 'schema.json')
  def execute_introspection_query
    # Cache the query result
    Rails.cache.fetch checksum do
      AppNameSchema.execute GraphQL::Introspection::INTROSPECTION_QUERY
    end
  end

  def checksum
    files   = Dir['app/graphql/**/*.rb'].reject { |f| File.directory?(f) }
    content = files.map { |f| File.read(f) }.join
    Digest::SHA256.hexdigest(content).to_s
  end

  def dump_schema
    # Generate the schema on start/reload
    FileUtils.mkdir_p SCHEMA_DIR
    result = JSON.pretty_generate(AppNameSchema.execute_introspection_query)
    unless File.exist?(SCHEMA_PATH) && File.read(SCHEMA_PATH) == result
      File.write(SCHEMA_PATH, result)
    end
  end
end

AppNameSchema.extend RelaySchemaHelpers
```

1. Add the following to .babelrc
```json
[
  "relay",
  {
    "schema": "app/javascript/packs/schema.json",
    "debug": true
  }
],
```

1. add `__generated__/` to `.gitignore`
1. create a `Procfile` with the following contents

```
web: bundle exec puma -t 5:5 -p ${PORT:-3000} -e ${RACK_ENV:-development}
relay: yarn run relay -- --watch
webpacker: ./bin/webpack-dev-server
```

1. add `"relay": "relay-compiler --src ./app/javascript --schema ./app/javascript/packs/schema.json"` as a script in `package.json`
1. create a controller action and in the view you can now render your `hello_react` component as `<%= react_component 'hello_react' %>`
