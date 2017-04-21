==================
 Extension Grants
==================

.. todo:: Describe how to implement extension grants. For now, take a look at one of the `existing grant types`_.

.. _existing grant types: https://github.com/oauthjs/node-oauth2-server/blob/master/lib/grant-types

Extension grants are registered through the ``options.extendedGrantTypes`` server option (see :ref:`OAuth2Server#token() <OAuth2Server#token>`).

This examples implements an extension grant (``MyGrantType``) that accepts or denies token requests depending on whether the client wants the request to succeed or fail. Of course this is not something you want to do in a real application.

::

  const OAuth2Server = require('oauth2-server');
  const AbstractGrantType = OAuth2Server.AbstractGrantType;
  const InvalidArgumentError = OAuth2Server.InvalidArgumentError;
  const Promise = require('bluebird');
  const promisify = require('promisify-any').use(Promise);

  function MyGrantType(options) {
    if (!options.model) {
      throw new InvalidArgumentError('Missing parameter: `model`');
    }

    // Check that the model implements all function this grant type needs.
    if (!options.model.getUser) {
      throw new InvalidArgumentError('Invalid argument: model does not implement `getUser()`');
    }

    if (!options.model.saveToken) {
      throw new InvalidArgumentError('Invalid argument: model does not implement `saveToken()`');
    }

    AbstractGrantType.call(this, options);
  }

  MyGrantType.prototype = Object.create(AbstractGrantType.prototype, {
    constructor: {value: MyGrantType, configurable: true, writable: true}
  });

  MyGrantType.prototype.handle = function handle(request, client) {
    if (!request) {
      throw new InvalidArgumentError('Missing parameter: `request`');
    }

    if (!response) {
      throw new InvalidArgumentError('Missing parameter: `response`');
    }

    let scope = this.getScope(request);

    return Promise.bind(this)
      .then(function() {
        return this.getUser(request);
      })
      .then(function(user) {
        return this.saveToken(user, client, scope);
      });
  };

  MyGrantType.prototype.getUser = function getUser(request) {
    // Verify `username` argument.
    if (!request.body.username) {
      throw new InvalidRequestError('Missing parameter: `username`');
    }

    // Verify `succeed` argument.
    if (!request.body.succeed) {
      throw new InvalidRequestError('Missing parameter: `succeed`');
    }

    if (['yes', 'no'].indexOf(request.body.succeed) < 0) {
      throw new InvalidRequestError('Invalid parameter: `succeed` must be \'yes\' or \'no\'');
    }

    // Let the request fail if the client wants it to.
    if (request.body.succeed !== 'yes') {
      throw new InvalidGrantError('Invalid grant: client chose to let the request fail');
    }

    // Call the model function `getUser()` without a password. This requires
    // that `getUser()` actually returns the user without verifying the password
    // if `null` is specified.
    return promisify(this.model.getUser, 2)(request.body.username, null)
      .then(function(user) {
        if (!user) {
          throw new InvalidGrantError('Invalid grant: user does not exist');
        }

        return user;
      });
  };

  MyGrantType.prototype.saveToken = function saveToken(user, client, scope) {
    // Validate the scope and generate access and refresh tokens.
    let fns = [
      this.validateScope(user, client, scope),
      this.generateAccessToken(client, user, scope),
      this.generateRefreshToken(client, user, scope),
      this.getAccessTokenExpiresAt(),
      this.getRefreshTokenExpiresAt()
    ];

    return Promise.all(fns)
      .bind(this)
      .spread(function(scope, accessToken, refreshToken, accessTokenExpiresAt, refreshTokenExpiresAt) {
        // The scope is valid. Save the generated tokens.
        let token = {
          accessToken,
          accessTokenExpiresAt,
          refreshToken,
          refreshTokenExpiresAt,
          scope
        };

        return promisify(this.model.saveToken, 3)(token, client, user);
      });
  };

In order to enable the server to use this grant, it has to be registered with the server instance:

::

  const OAuth2Server = require('oauth2-server');

  let oauth = new OAuth2Server({
    // ...
    extendedGrantTypes: {
      'my_grant': MyGrantType
    }
  });

