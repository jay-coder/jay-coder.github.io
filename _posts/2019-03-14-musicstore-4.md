---
layout: post
title: MusicStore - Part4 - External Login
---

Let's enable the external login, so that Microsoft and LinkedIn users don't need to sign up in Music Store.

1. Microsoft

    * Create an app in Microsoft via [https://apps.dev.microsoft.com/](https://apps.dev.microsoft.com/)
      - Fill in Name
      - Generate New Password
      <img src='{{ "/public/assets/img/musicstore_ms_app_newpass.png" | relative_url }}' alt="Generate New Password" />

      - Add Platform
      <img src='{{ "/public/assets/img/musicstore_ms_app_add.png" | relative_url }}' alt="Add Microsoft App" />

      <p class="message">
        NOTE: Redirect URLs need to be consistent with CallbackPath in Startup.cs, otherwise, you will get an error: <br><strong>The provided value for the input parameter 'redirect_uri' is not valid</strong>
      </p>
    * Update IdentityServer4 to authenticate Microsoft users

      - Install NuGet Package - Microsoft.AspNetCore.Authentication.MicrosoftAccount
      - Add Microsoft Authentication

      ```csharp
      // Startup.cs
      public void ConfigureServices(IServiceCollection services)
      {
          ......
        
          services.AddAuthentication()
            .AddMicrosoftAccount(options =>
            {
              options.SignInScheme = IdentityServerConstants.ExternalCookieAuthenticationScheme;

              options.ClientId = Configuration["Microsoft:ApplicationId"];
              options.ClientSecret = Configuration["Microsoft:Password"];

              options.CallbackPath = new PathString("/microsoft");
              options.SaveTokens = true;
            });
          ......
      }

      public void Configure(IApplicationBuilder app)
      {
          ......
          app.UseAuthentication();
          ......
      }
      ```
      
      - Store Application Id and Password from #1.1 to appsettings.Development.json
    
      ```json
      {
        ......
        "Microsoft": {
          "ApplicationId": "0d1c72bb-ec52-4a0c-a582-4ad79f9a9c5a",
          "Password": "vnssBKZWR237*ujtZW44@*|"
        }
        ......
      }
      ```

2. LinkedIn
  
    <p class="message">NOTE: LinkedIn upgraded their API recently, it won't carry back user email, a key field in ApplicationUser, so LinkedIn user could login.</p>

    * Create an app in LinkedIn via [https://www.linkedin.com/developers/apps](https://www.linkedin.com/developers/apps)

      - Fill in App name, Company name, App description, Logo, and Business email
        <img src='{{ "/public/assets/img/musicstore_li_app_add.png" | relative_url }}' alt="Add LinkedIn App" />

    * Update IdentityServer4 to authenticate LinkedIn users
      - Install NuGet Package - AspNet.Security.OAuth.LinkedIn
      - Add LinkedIn Authentication

        ```csharp
        public void ConfigureServices(IServiceCollection services)
        {
            ......
          
            services.AddAuthentication()
              .AddLinkedIn(options =>
              {
                options.SignInScheme = IdentityServerConstants.ExternalCookieAuthenticationScheme;

                options.ClientId = Configuration["LinkedIn:ClientId"];
                options.ClientSecret = Configuration["LinkedIn:ClientSecret"];

                options.CallbackPath = new PathString("/linkedin");
                options.SaveTokens = true;
              });
            ......
        }
        ```

      - Store Client Id and Client Secret from #2.1 to appsettings.Development.json

        ```json
        {
          ......
          "LinkedIn": {
            "ClientId": "816crn5hi7uoxx",
            "ClientSecret": "lyV15uviOnli9tQr"
          }
          ......
        }
        ```

3. Authenticate users from external OAuth 2.0 providers

    * Replace TestUserStore with AspNetIdentity UserManager and SignInManager

      ```csharp
      // ExternalController.cs
      private readonly UserManager<ApplicationUser> _userManager;
      private readonly SignInManager<ApplicationUser> _signInManager;
      public ExternalController(
        IIdentityServerInteractionService interaction,
        IClientStore clientStore,
        IEventService events,
        UserManager<ApplicationUser> userManager,
        SignInManager<ApplicationUser> signInManager
      )
      {
        _interaction = interaction;
        _clientStore = clientStore;
        _events = events;

        _userManager = userManager;
        _signInManager = signInManager;
      }
      ```

    * External login callback

        ```csharp
        // ExternalController.cs
        [HttpGet]
        public async Task<IActionResult> Callback()
        {
          // read external identity from the temporary cookie
          var result = await HttpContext.AuthenticateAsync(IdentityServer4.IdentityServerConstants.ExternalCookieAuthenticationScheme);
          if (result?.Succeeded != true)
          {
            throw new Exception("External authentication error");
          }

          var extPrincipal = result.Principal;
          var expProperties = result.Properties;
          var claims = extPrincipal.Claims.ToList();

          var userIdClaim = claims.FirstOrDefault(x => x.Type == JwtClaimTypes.Subject);
          if (userIdClaim == null)
          {
            userIdClaim = claims.FirstOrDefault(x => x.Type == ClaimTypes.NameIdentifier);
          }
          if (userIdClaim == null)
          {
            throw new Exception("Unknown userid");
          }

          claims.Remove(userIdClaim);
          var provider = expProperties.Items["scheme"];
          var userId = userIdClaim.Value;

          var user = await _userManager.FindByLoginAsync(provider, userId);
          if (user == null)
          {
                        // Register if user doesn't exist
            var candidateId = Guid.NewGuid();
            user = new ApplicationUser();
            var emailClaim = claims.FirstOrDefault(x => x.Type == ClaimTypes.Email);
            if (emailClaim != null)
            {
              user.UserName = emailClaim.Value;
              user.Email = emailClaim.Value;
              user.EmailConfirmed = true;
            }
            var firstNameClaim = claims.FirstOrDefault(x => x.Type == ClaimTypes.GivenName);
            if (firstNameClaim != null)
            {
              user.FirstName = firstNameClaim.Value;
            }
            var lastNameClaim = claims.FirstOrDefault(x => x.Type == ClaimTypes.Surname);
            if (lastNameClaim != null)
            {
              user.LastName = lastNameClaim.Value;
            }
            user.UserType = EnumUserType.Audience;
            var createResult = await _userManager.CreateAsync(user);
            if (createResult.Succeeded)
            {
              var loginResult = await _userManager.AddLoginAsync(user, new UserLoginInfo(provider, userId, provider));
              if (loginResult.Succeeded)
              {
                // Update Tokens to AspNet Identity
                var externalLoginInfo = new ExternalLoginInfo(extPrincipal, provider, userId, provider)
                {
                  AuthenticationTokens = expProperties.GetTokens()
                };
                await _signInManager.UpdateExternalAuthenticationTokensAsync(externalLoginInfo);
              }
            }
          }
          ......
        }
        ```

        <img src='{{ "/public/assets/img/musicstore_is_ext_login.png" | relative_url }}' alt="External Login" />

4. Display user information in WebUI
  
    * Update welcome message in nav-menu component

      ```html
      <pre class="wp-block-code"><code>// nav-menu.component.html
      <div class='main-nav'>
        <div class='navbar navbar-inverse'>
          <div class='navbar-header'>
        ......
        <a class='navbar-brand' [routerLink]='["/"]'>Welcome {{firstName}} {{lastName}}</a>
          </div>
          ......
        </div>
      </div>
      ```

      ```js
      // nav-menu.component.ts
      import { Component } from '@angular/core';
      import { UserProfileService } from '../userprofile/userprofile.service';

      @Component({
        selector: 'app-nav-menu',
        templateUrl: './nav-menu.component.html',
        styleUrls: ['./nav-menu.component.css']
      })
      export class NavMenuComponent {
        isExpanded = false;
        firstName: string = '';
        lastName: string = '';
        constructor(
          private _userProfileService: UserProfileService
        ) {
          var that = this;
          this._userProfileService.identityClaimsReady.subscribe(function (claims) {
            if (claims) {
              that.firstName = claims["FirstName"];
              that.lastName = claims["LastName"];
            }
          });
        }
        ......
      }
      ```

    * Send claims to subscribers

      ```js
      // app.component.ts
      ......
      import { UserProfileService } from './userprofile/userprofile.service';

      @Component({
        selector: 'app-root',
        templateUrl: './app.component.html',
        styleUrls: ['./app.component.css']
      })
      export class AppComponent implements OnInit {
        title = 'app';
        constructor(
          @Inject(PLATFORM_ID) private platformId: Object,
          private _oauthService: OAuthService,
          private _userProfileService: UserProfileService
        ) {
          ......
        }
        /**
          * On init
          */
        ngOnInit(): void {
          if (isPlatformBrowser(this.platformId)) {
            this._oauthService.loadDiscoveryDocumentAndTryLogin().then(_ => {
              if (!this._oauthService.hasValidIdToken() || !this._oauthService.hasValidAccessToken()) {
                this._oauthService.initImplicitFlow();
              } else {
                this._userProfileService.onIdentityClaimsReadyChanged(this._oauthService.getIdentityClaims());
              }
            });
          }
        }
      }
      ```

    * Add userprofile.service.ts to observe claim changes

      ```js
      import { Injectable } from '@angular/core';
      import { Subject } from 'rxjs';

      @Injectable()
      export class UserProfileService {
        identityClaimsReady: Subject<any> = new Subject();

        onIdentityClaimsReadyChanged(claims): void {
          this.identityClaimsReady.next(claims);
        }
      }
      ```

5. Include FirstName and LastName in Claims from IdentityServer4

    ```csharp
    // ProfileService.cs
    using IdentityServer4.Models;
    using IdentityServer4.Services;
    using JayCoder.MusicStore.Core.Domain.SQLEntities;
    using Microsoft.AspNetCore.Identity;
    using System.Collections.Generic;
    using System.Security.Claims;
    using System.Threading.Tasks;

    namespace JayCoder.MusicStore.Projects.IdentityServer.Profile
    {
        public class ProfileService : IProfileService
        {
            protected UserManager<ApplicationUser> _userManager;

            public ProfileService(UserManager<ApplicationUser> userManager)
            {
                _userManager = userManager;
            }

            public Task GetProfileDataAsync(ProfileDataRequestContext context)
            {
                var user = _userManager.GetUserAsync(context.Subject).Result;
                if (user != null)
                {
                    var claims = new List<Claim>();
                    claims.Add(new Claim("FirstName", string.IsNullOrEmpty(user.FirstName) ? string.Empty : user.FirstName));
                    claims.Add(new Claim("LastName", string.IsNullOrEmpty(user.LastName) ? string.Empty : user.LastName));
                    context.IssuedClaims.AddRange(claims);
                }
                return Task.FromResult(0);
            }
            ......
        }
    }
    ```

    <p class="message">
      NOTE: In order to access user claims from Angular, <strong>AlwaysIncludeUserClaimsInIdToken</strong> needs to be true when you register Client in IdentityServer4.
    </p>

6. Test

    Login from my Microsoft account and be able to access values API
    <img src='{{ "/public/assets/img/musicstore_ms_test.png" | relative_url }}' alt="Test Microsoft Account Login" />

