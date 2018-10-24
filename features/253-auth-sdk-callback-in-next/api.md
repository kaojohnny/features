# Description

In skygear v1, auth related API in SDK often returns a `user<record>` as the result. e.g.

```
skygear.auth.signupWithUsername(username, password).then((user) => {
  console.log(user); // user is an user record
  console.log(user["username"]); // username of the user
}, (error) => {
  ...
});
```

and user can update its profile via `record` API, like:

```
var user = skygear.auth.currentUser;
user["username"] = "new-username";
skygear.publicDB.save(user).then((user) => {
  console.log(user); // updated user record
  console.log('Username is changed to: ', user["username"]);
  return skygear.auth.whoami();
}, (error) => {
  console.error(error);
});
```

---

In skygear next, we will have several gears, and each gear handles its own responsibility only. For user auth related functionalities, we will use two gears to handle a user's state.

- auth gear
    - It handles auth related information:
        - user role, disable state, verify state, etc.
        - principal (how a user can login, e.g. (username/email) + password)
- record gear (optional)
    - A general purpose gear to handle user profile
    - Accessible by record gear only
    - Expected to be accessed by Auth gear through public API optionally

There are some concerns arisen due to the design:

1. since record gear is optional, what is the expected return data for the auth related API?
2. how does a user update his own profile? especially when he tries to update his username or email that connects to auth data.

Followings are some concerns are known but will not be discussed in this PR.

1. should auth gear support query functionality?
2. what is the expected parameter data type when admin updates other users' info?

# Scenario

- [user] sign up and login
- [user] access auth data (roles, disabled)
- [user] access user profile (username, email, phone number, ...)
- [user] update user profile (username, email, phone number, ...)

skip following admin scenarios temporarily:
- ~~[admin] query other users by some predicates~~
- ~~[admin] set other users auth data (roles, disabled)~~

# Changes on SDK

We consider 3 options mainly:

   - [**option1**] profile record is part of `currentUser`
     
     ```
     skygear.auth.currentUser; // core user + user record
     skygear.auth.currentUser.profile; // user record
     
     // auth info
     skygear.auth.currentUser.disabled;
     skygear.auth.currentUser.verified;
     
     // profile info
     skygear.auth.currentUser.profile['phone'];
     
     // update record
     skygear.auth.currentUser['username'] = 'new-username';
     skygear.auth.currentUser.profile['phone'] = 'new-phone';
     skygear.auth.updateUser(skygear.auth.currentUser);
     
     // it is uncertain for now if developer saves via record gear
     skygear.auth.currentUser.profile['username'] = 'new-username';
     skygear.publicDB.save(skygear.auth.currentUser.profile); // ??? what should happened? should record save back to auth gear?
     ```
     
     Pros:
     - easy to implement.
     - it is clear that we have two ideas here: core user and user record.
     
     Cons:
     - it still conveys two ideas, core user and user record, it may confuse a developer to use which attribute when invokes auth related operations, e.g. when a user wants to update username, which one should I update? `skygear.auth.currentUser['username']` or `skygear.auth.currentUser.profile['username']`?
     - not sure how to handle record save directly problem.

   - [**option2**] keep current design, return is a `record`
      
     ```
     skygear.auth.currentUser; // user record
     
     // auth info
     skygear.auth.isCurrentUserVerified;
     skygear.auth.isCurrentUserDisabled;
     
     // profile info
     const phone = skygear.auth.currentUser['phone'];
     
     // update record
     skygear.auth.currentUser['username'] = 'new-username';
     skygear.auth.currentUser['phone'] = 'new-phone';
     skygear.auth.updateUser(skygear.auth.currentUser);
     
     // it is uncertain for now if developer saves via record gear
     skygear.auth.currentUser['username'] = 'new-username';
     skygear.publicDB.save(skygear.auth.currentUser.profile); // ??? what should happened?
     ```
     Pros:
     - almost sync with current design.
     
     Cons:
     - if record gear doesn't configured, it could be very weired if API still return a record object.
     - since auth gear may need handle record save and implement `updateUser(<record>)` endpoint, auth gear will be coupled with record gear.
     - not sure how to handle record save directly problem.

   - [**option3**] no user record, SDK will wrap it to a new `SKUser` type
     
     ```
     skygear.auth.currentUser; // SKUser type
     
     // auth info
     skygear.auth.currentUser.disabled;
     skygear.auth.currentUser.verified;
     
     // profile info
     const phone = skygear.auth.currentUser['phone'];
     
     // update record
     skygear.auth.currentUser['username'] = 'new-username';
     skygear.auth.currentUser['phone'] = 'new-phone';
     skygear.auth.updateUser(skygear.auth.currentUser);
     ```
     
     Pros:
     
     - developer won't confused with record save.
     - it is clear to use `updateUser` API.
     - it should be fine if record gear is not configured.


     Cons:
     - introduce a new data type
     - a developer may update reserved attributes, like `skygear.auth.currentUser.disabled = false`.
     - (Ben) could be a lot of edge case especially when Cloud functions and user querying user table happens.