# Merit Gem: Reputation rules (badges, points and rankings) for Rails applications

![Merit](http://i567.photobucket.com/albums/ss118/DeuceBigglebags/th_nspot26_300.jpg)

[![Build Status](https://travis-ci.org/tute/merit.png?branch=master)](http://travis-ci.org/tute/merit)

# Installation

1. Add `gem 'merit'` to your `Gemfile`
2. Run `rails g merit:install`
3. Run `rails g merit MODEL_NAME` (e.g. `user`)
4. Run `rake db:migrate`
5. Define badges you will use in `config/initializers/merit.rb`
6. Configure reputation rules for your application in `app/models/merit/*`

---

# Defining badge rules

You may give badges to any resource on your application if some condition
holds. Badges may have levels, and may be temporary. Define rules on
`app/models/merit/badge_rules.rb`:

`grant_on` accepts:

* `'controller#action'` string (similar to Rails routes)
* `:badge` for badge name
* `:level` for badge level
* `:to` method name over target_object which obtains object to badge
* `:model_name` (string) define controller's name if it differs from
  the model (like `RegistrationsController` for `User` model).
* `:multiple` (boolean) badge may be granted multiple times
* `:temporary` (boolean) if the receiver had the badge but the condition
  doesn't hold anymore, remove it. `false` by default (badges are kept
  forever).
* `&block`
  * empty (always grants)
  * a block which evaluates to boolean (recieves target object as parameter)
  * a block with a hash composed of methods to run on the target object with
    expected values

## Examples

```ruby
grant_on 'comments#vote', :badge => 'relevant-commenter', :to => :user do |comment|
  comment.votes.count == 5
end

grant_on ['users#create', 'users#update'], :badge => 'autobiographer', :temporary => true do |user|
  user.name.present? && user.address.present?
end
```

## Grant manually

You may also grant badges "by hand" (optionally multiple times):

```ruby
Badge.find(3).grant_to(current_user, :allow_multiple => true)
```

Similarly you can take back badges that have been granted:

```ruby
Badge.find(3).delete_from(current_user)
```

---

# Defining point rules

Points are given to "meritable" resources on actions-triggered, either to the
action user or to the method(s) defined in the `:to` option. Define rules on
`app/models/merit/point_rules.rb`:

`score` accepts:

* `:on` action as string or array of strings (similar to Rails routes)
* `:to` method(s) to send to the target_object (who should be scored?)
* `&block`
  * empty (always scores)
  * a block which evaluates to boolean (recieves target object as parameter)

## Examples

```ruby
score 10, :to => :post_creator, :on => 'comments#create' do |comment|
  comment.title.present?
end

score 20, :on => [
  'comments#create',
  'photos#create'
]

score 15, :on => 'reviews#create', :to => [:reviewer, :reviewed]
```

## Score manually

You may also change user points "by hand":

```ruby
current_user.add_points(20, 'Optional log message')

current_user.substract_points(10)
```

---

# Defining rank rules

5 stars is a common ranking use case. They are not given at specified actions
like badges, you should define a cron job to test if ranks are to be granted.

Define rules on `app/models/merit/rank_rules.rb`:

`set_rank` accepts:

* `:level` ranking level (greater is better)
* `:to` model or scope to check if new rankings apply
* `:level_name` attribute name (default is empty and results in
  '`level`' attribute, if set it's appended like
  '`level_#{level_name}`')

Check for rules on a rake task executed in background like:

```ruby
task :cron => :environment do
  Merit::RankRules.new.check_rank_rules
end
```


## Examples

```ruby
set_rank :level => 2, :to => Commiter.active do |commiter|
  commiter.branches > 1 && commiter.followers >= 10
end

set_rank :level => 3, :to => Commiter.active do |commiter|
  commiter.branches > 2 && commiter.followers >= 20
end
```

---

# To-do list

* `Merit::BadgeRules.new.defined_rules` should be cached on initialization,
  instead of initialized per controllers `after_filter` and
  `merit_action.check_rules`.
* target_object should be configurable (now it's singularized controller name)
* Translate comments from spanish in `rules_badge.rb`.
* Should namespace app/models into Merit module.
* :value parameter (for star voting for example) should be configurable
  (depends on params[:value] on the controller).
