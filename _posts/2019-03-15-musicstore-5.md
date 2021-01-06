---
layout: post
title: MusicStore – Part5 – Authorization
redirect_from:
  - /musicstore-5/
---

Let's authorize users by their types - ONLY "Musician" can access to Values API.

1. Include UserType in claims in IdentityServer4

    ```csharp
    // ProfileService.cs
    public Task GetProfileDataAsync(ProfileDataRequestContext context)
    {
      var user = _userManager.GetUserAsync(context.Subject).Result;
      if (user != null)
      {
        var claims = new List&lt;Claim>();
        claims.Add(new Claim(AuthorizeClaim.FirstName, string.IsNullOrEmpty(user.FirstName) ? string.Empty : user.FirstName));
        claims.Add(new Claim(AuthorizeClaim.LastName, string.IsNullOrEmpty(user.LastName) ? string.Empty : user.LastName));
        claims.Add(new Claim(AuthorizeClaim.UserType, user.UserType.ToString()));
        context.IssuedClaims.AddRange(claims);
      }
      return Task.FromResult(0);
    }
    ```

2. Add authorization policies in WebAPI

    Policies can be built by claim, role, scope or assertion with conditions.
    ```csharp
    // Startup.cs
    public void ConfigureServices(IServiceCollection services)
    {
      var svcBuilder = services
        .AddMvcCore()
        .AddApiExplorer()
        .AddJsonFormatters();

      svcBuilder = svcBuilder.AddAuthorization(
        options => BuildAuthorizationPolicies(options)
      );
      ......
    }

    private void BuildAuthorizationPolicies(AuthorizationOptions options)
    {
      //Basic Policies
      foreach (var userType in Enum.GetNames(typeof(EnumUserType)))
      {
        options.AddPolicy(userType, policy => policy.RequireClaim(AuthorizeClaim.UserType, userType));
      }
    }
    ```

3. Add policy to ValuesController to ONLY allow "Musician"

    ```csharp
    [Authorize(Policy = AuthorizePolicy.Musician)]
    [Route("api/[controller]")]
    [ApiController]
    public class ValuesController : ControllerBase
    {
        ......
    }
    ```

4. Test

    Login with Microsoft account will return 403 when clicking on Sample, as external users are "Audience".

    However, Alice can get results from Values API, because she belongs to "Musician".

5. Lock down permissions for WebUI

    * Add authguard.service.ts to validate UserType from claims

        ```js
        // authguard.service.ts
        import { Injectable } from '@angular/core';
        import { ActivatedRouteSnapshot, CanActivate, Router, RouterStateSnapshot } from '@angular/router';
        import { OAuthService } from 'angular-oauth2-oidc';

        @Injectable()
        export class AuthGuardService implements CanActivate {
            constructor(
                private _oauthService: OAuthService,
                private _router: Router
            )
            {
            }

            canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): boolean {
                var isRoleValid = this._oauthService.hasValidIdToken();
                if (isRoleValid)
                {
                    var expectedRole = route.data.expectedRole;
                    var claims = this._oauthService.getIdentityClaims();
                    if (claims &amp;&amp; expectedRole) {
                        if (expectedRole.indexOf(",") >= 0) {
                            var roleArray = expectedRole.split(",");
                            isRoleValid = roleArray.indexOf(claims["UserType"]) >= 0
                        }
                        else {
                            isRoleValid = claims["UserType"] == expectedRole;
                        }
                    }
                }
                if (!isRoleValid) {
                  alert("Sorry, you don't have access to this module!");
                  this._router.navigate(['/']);
                }
                return isRoleValid;
            }
        }
        ```

    * Apply AuthGuard service to validate role for sample component
        
        ```js
        // app.module.ts
        import { BrowserModule } from '@angular/platform-browser';
        import { NgModule } from '@angular/core';
        import { FormsModule } from '@angular/forms';
        import { HttpClientModule, HTTP_INTERCEPTORS } from '@angular/common/http';
        import { RouterModule } from '@angular/router';

        //OAuth2 OIDC Client
        import { OAuthModule } from 'angular-oauth2-oidc';
        //Components
        import { AppComponent } from './app.component';
        import { NavMenuComponent } from './nav-menu/nav-menu.component';
        import { HomeComponent } from './home/home.component';
        import { CounterComponent } from './counter/counter.component';
        import { FetchDataComponent } from './fetch-data/fetch-data.component';
        import { SampleComponent } from './sample/sample.component';
        //Http Interceptor
        import { TokenInterceptor } from './http/token.interceptor';
        //Services
        import { UserProfileService } from './userprofile/userprofile.service';
        //Security
        import { AuthGuardService } from './security/authguard.service';

        @NgModule({
          declarations: [
            AppComponent,
            NavMenuComponent,
            HomeComponent,
            CounterComponent,
            FetchDataComponent,
            SampleComponent
          ],
          imports: [
            BrowserModule.withServerTransition({ appId: 'ng-cli-universal' }),
            HttpClientModule,
            FormsModule,
            OAuthModule.forRoot(),
            RouterModule.forRoot([
              { path: '', component: HomeComponent, pathMatch: 'full' },
              { path: 'counter', component: CounterComponent },
              { path: 'fetch-data', component: FetchDataComponent },
              {
                path: 'sample',
                component: SampleComponent,
                canActivate: [AuthGuardService],
                data: {
                  expectedRole: 'Musician'
                }
              }
            ])
          ],
          providers: [
            AuthGuardService,
            UserProfileService,
            {
              provide: HTTP_INTERCEPTORS,
              useClass: TokenInterceptor,
              multi: true,
            }
          ],
          bootstrap: [AppComponent]
        })
        export class AppModule { }
        ```

6. Test again

    Login as Microsoft account will see this error message, as it's been blocked by AuthGuard from front-end part.

    <img src='{{ "/public/assets/img/musicstore_test_role.png" | relative_url }}' alt="Test User Role" />