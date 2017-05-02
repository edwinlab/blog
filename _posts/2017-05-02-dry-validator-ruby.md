---
layout: post
title: "Simple validator ruby"
description: "Simple dry validator in ruby."
tags: [ruby, validation]
---
# Simple validator ruby
Dry validator in ruby to validate parameter.

```ruby
module Validate
  class Activity
    include ActiveModel::Validations

    attr_accessor :latitude, :longitude

    validates :latitude, presence: true, numericality: true
    validates :longitude, presence: true, numericality: true

    def initialize(params={})
      @latitude  = params[:latitude]
      @longitude = params[:longitude]
      ActionController::Parameters.new(params).permit(:latitude,:longitude)
    end
  end
end
```

And implementation in controllers

```ruby
class LocationsController < ApplicationController
  before_action :validate_params

  def index
  end

  rescue_from(ActionController::UnpermittedParameters) do |pme|
    render json: { error:  { unknown_parameters: pme.params } },
               status: :bad_request
  end

  private
  def validate_params
    location = Validate::Location.new(params)
    if !location.valid?
      render json: { error: location.errors } and return
    end
  end
end
```