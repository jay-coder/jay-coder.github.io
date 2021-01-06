---
layout: post
title: MusicStore - Part2 - Link three parties
redirect_from:
  - /musicstore-2/
---

## Register Client in IdentityServer

```csharp
"Clients": [
    {
      "ClientId": "portal-client",
      "ClientName": "MusicStore Portal Client",
      "ClientUri": "http://localhost:30002",
      "RequireConsent": false,
      "AllowedGrantTypes": "implicit",
      "AllowAccessTokensViaBrowser": true,
      "RedirectUris": "http://localhost:30002",
      "PostLogoutRedirectUris": "http://localhost:30002/loggedout",
      "AllowedCorsOrigins": "http://localhost:30002",
      "AllowedScopes": "openid;profile;email;musicstore-api",
      "AlwaysIncludeUserClaimsInIdToken": true,
      "IdentityTokenLifetime": 3600,
      "AccessTokenLifetime": 3600
    }
  ]
```

1. Create a new Client called "portal-client"

    * ClientUri => WebUI Uri
    * RequireConsent => <strong>false</strong> means disable consent dialog after login
    * AllowedGrantTypes => Grant type of SPA client needs to be <strong>implicit</strong>
    * RedirectUris => Redirect Uri after login, id_token will be appended 
    * PostLogoutRedirectUris => Redirect Uri after logging out 
    * AllowedCorsOrigins => WebUI Uri
    * AllowedScopes => API scopes can be consumed for this client

2. Register musicstore-api

    ```csharp
    // Configuration/Resources.cs
    public static IEnumerable<ApiResource> GetApiResources()
    {
        return new[]
        {
            ......
            new ApiResource("musicstore-api", "MusicStore API")
        };
    }
    ```

## Authenticate WebAPI from IdentityServer

1. Add Authentication from IdentityServer 
2. Add CORS to allow origins from WebUI

    ```csharp
    // Startup.cs
    public void ConfigureServices(IServiceCollection services)
    {
      services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_1);
            // Authenticated by IdentityServer
      services
        .AddAuthentication("Bearer")
        .AddIdentityServerAuthentication(options =>
        {
          options.Authority = Configuration.GetValue<string>("Authority");
          options.RequireHttpsMetadata = Configuration.GetValue<bool>("RequireHttpsMetadata");
          options.ApiName = Configuration.GetValue<string>("ApiName");
        });
            // CORS Policy
      string strOrigionList = Configuration.GetValue<string>("AllowedOrigions");
      if (!string.IsNullOrEmpty(strOrigionList))
      {
        services.AddCors(options =>
        {
          // this defines a CORS policy called "default"
          options.AddPolicy("default", policy =>
          {
            policy.WithOrigins(strOrigionList.Split(new string[] { ";" }, StringSplitOptions.RemoveEmptyEntries))
              .AllowAnyHeader()
              .AllowAnyMethod();
          });
        });
      }
    }
    ```
    ```csharp
    // Startup.cs
    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
      if (env.IsDevelopment())
      {
        app.UseDeveloperExceptionPage();
      }
      app.UseCors("default");
      app.UseAuthentication();

      app.UseMvc();
    }
    ```
    ```json
    // appsettings.Development.json
    {
      "Logging": {
        "LogLevel": {
          "Default": "Debug",
          "System": "Information",
          "Microsoft": "Information"
        }
      },
      // Authentication - IdentityServer
      "Authority": "http://localhost:30000",
      "RequireHttpsMetadata": false,
      "ApiName": "musicstore-api",
      // CORS - Allow WebUI
      "AllowedOrigions": "http://localhost:30002"
    }
    ```
3. Disable anonymous access to values controller

    ```csharp
    // ValuesController.cs
    [Authorize]
    [Route("api/[controller]")]
    [ApiController]
    public class ValuesController : ControllerBase
    {
        ......
    }
    ```

4. Test from Postman
    * Select "TYPE" as "OAuth 2.0" in Authorization tab
    * Click "Get New Access Token"
    * Fill in the Client info and click "Request Token"

      <img src='{{ "/public/assets/img/musicstore_get_token.png" | relative_url }}' alt="Postman Get JWT Token" />

    * Username / Password - bob / bob or alice / alice
    * After getting the JWT token, attach it to Header and fire "GET" request to http://localhost:30001/api/values will return ["value1", "value2"]

      <img src='{{ "/public/assets/img/musicstore_get_values.png" | relative_url }}' alt="Postman Get Values" />

## Consume WebAPI from WebUI

1. Install OIDC to bridge WebUI and IdentityServer

    ```shell
    npm install angular-oauth2-oidc@^3 --save
    ```

    <p class="message">
      NOTE: As our WebUI utilized Angular 5.2, it could not be compatible with the latest oidc (version 4), otherwise, it will throw this error:<br>
      <strong>ERROR TypeError: Object(…) is not a function </strong>
    </p>

2. Apply OIDC

    * Update AppModule to include OAuthModule
      ```js
      @NgModule({
        declarations: [
          ......
        ],
        imports: [
          ......
          OAuthModule.forRoot()
        ],
        providers: [
          {
            provide: HTTP_INTERCEPTORS,
            useClass: TokenInterceptor,
            multi: true,
          }
        ]
        bootstrap: [AppComponent]
      })
      export class AppModule { }
      ```
    
    * Add AuthConfig

      ```js
      export const authConfig: AuthConfig = {
        // Url of the Identity Provider
        issuer: environment.identityServerUrl,

        // URL of the SPA to redirect the user to after login
        redirectUri: window.location.origin,

        // The SPA's id. The SPA is registerd with this id at the auth-server
        clientId: 'portal-client',

        // set the scope for the permissions the client should request
        // The first three are defined by OIDC. The 4th is a usecase-specific one
        scope: 'openid profile email ' + environment.apiScope,
        postLogoutRedirectUri: environment.identityServerUrl + '/account/login'
      }
      ```

      <p class="message">NOTE: AuthConfig needs to be consistent with the client registered in IdentityServer e.g. clientId, scope.</p>

    * Environment configuration
      ```js
      // environment.ts
      export const environment = {
        production: false,

        apiScope: "musicstore-api",
        apiUrl: 'http://localhost:30001/api/',
        identityServerUrl: 'http://localhost:30000'
      };
      ```
      
      ```js
      // environment.prod.ts
      export const environment = {
        production: true,

        apiScope: "musicstore-api",
        apiUrl: 'http://api.jaycoder.net/api/',
        identityServerUrl: 'https://id.jaycoder.net'
      };
      ```

      <p class="message">
        NOTE: environment.ts has the configuration for development environment, while environment.prod.ts is for production, environment.ts will be replaced by environment.prod.ts if <strong>env</strong> argument is <strong>prod</strong> at <strong>build</strong> stage.
      </p>

      ```shell
      ng build --env=prod
      ```

    * Initialize OAuthService in AppComponent

      Token will be validated in ngOnInit method, if it's expired, initImplicitFlow will redirect to IdentityServer login to reissue a new one.

      ```js
      export class AppComponent {
        title = 'app';
        constructor(
          @Inject(PLATFORM_ID) private platformId: Object,
          private _oauthService: OAuthService
        ) {
          if (isPlatformBrowser(this.platformId)) {
            this._oauthService.configure(authConfig);
            this._oauthService.tokenValidationHandler = new JwksValidationHandler();
            
            if (!environment.production) {
              this._oauthService.events.subscribe(e => {
                console.log("oauth/oidc event", e);
              });
            }
          }
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

              }
            });
          }
        }
      }
      ```
3. Add a new component "Sample" to call /api/values
  ```js
  export class SampleComponent {
    public values: string[];

    constructor(http: HttpClient) {
      var valuesApiUrl = environment.apiUrl + 'values';
      http.get<string[]>(valuesApiUrl).subscribe(result => {
        this.values = result;
      }, error => console.error(error));
    }
  }
  ```

4. Add TokenInterceptor to append Bearer Authorization in header
    ```js
    export class TokenInterceptor implements HttpInterceptor {

        private _authService: OAuthService;

        // Would like to inject authService directly but it causes a cyclic dependency error
        // see https://github.com/angular/angular/issues/18224
        constructor(private _injector: Injector) { }

        intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
            if (this.getAuthService().hasValidAccessToken()) {
                request = request.clone({
                    // Remove cache in IE
                    headers: request.headers.set('Cache-Control', 'no-cache')
                        .set('Pragma', 'no-cache')
                        .set('Expires', 'Sat, 01 Jan 1900 00:00:00 GMT')
                        .set('Authorization', 'Bearer ' + this.getAuthService().getAccessToken())
                });
            }
            return next.handle(request);
        }

        getAuthService(): OAuthService {
            if (typeof this._authService === 'undefined') {
                this._authService = this._injector.get(OAuthService);
            }
            return this._authService;
        }
    }
    ```
    <p class="message">NOTE: TokenInterceptor needs to be registered as a provider in AppModule.</p>

## Test

<img src='{{ "/public/assets/img/musicstore_test_sample.png" | relative_url }}' alt="MusicStore Test Sample" />