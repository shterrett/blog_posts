# Test your seed data!

I grabbed someone else's project recently and went to set it up on my local machine. When I ran `rake db:seed`, I was in for a nasty surprise: the data failed validations and the seed didn't run.

It's easy to let seed data get out of date; usually a developer seeds data at the beginning of a project, and then doesn't need it again. As the project evolves, the data gets stale. Fortunately, it's also easy to keep this from happening, and make it easy for people to set up the project later: spec the seed data!

Here's how:

```
require 'spec_helper'
require 'rake'
require 'faker'

describe 'db:seed task' do
  it 'seeds the database with valid data' do
    Meritorious::Application.load_tasks
    expect do
      Rake::Task['db:seed'].invoke
    end.not_to raise_error
  end
end
```

The `require 'faker'` is in there because the excellent [faker gem](https://github.com/stympy/faker) is used to generate my seed data. `rake` also has to be required so that the task can be run. Then the test simply says 'seed my database without throwing an exception'. Super simple, and if changing business rules change validations and the seed data goes out of date, then it will be caught, and can be fixed, immediately. 

This is along the same lines as the excellent built in factory tests for [factory_girl](https://github.com/thoughtbot/factory_girl)

```
require 'spec_helper'

FactoryGirl.factories.map(&:name).each do |factory_name|
  describe "factory #{factory_name}" do
    it 'is valid' do
      factory = build(factory_name)

      if factory.respond_to?(:valid?)
        expect(factory).to be_valid, factory.errors.full_messages.join(',')
      end
    end
  end
end
```

They build an instance of each factory and ensure that it's valid, so that if the requirements ever change, the factories will break and can be changed, without forcing you to hunt the bugs down in your test suite.