@model ProformaApplication.ViewModel.LoginViewModel
           
@{
    ViewBag.Title = "Login";
    Layout = null;
}

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <title>Login</title>
    <link href="@Url.Content("~/Content/global.css")" rel="stylesheet" type="text/css" />
    <link href="@Url.Content("~/Content/ddmenu.css")" rel="stylesheet" type="text/css" />
    <link href="@Url.Content("~/Content/jquery-ui-1.9.1.custom.css")" rel="stylesheet" type="text/css" />
    <script type="text/javascript" src="@Url.Content("~/Scripts/jquery-1.8.2.min.js")"></script>
    <script type="text/javascript" src="@Url.Content("~/Scripts/modernizr.custom.2.0.2.js")"></script>
    <script type="text/javascript" src="@Url.Content("~/Scripts/jquery-ui.js")"></script>
    <script  type="text/javascript" src="@Url.Content("~/Scripts/jquery.validate.js")"></script>
    <script type="text/javascript" src="@Url.Content("~/Scripts/jquery.validate.unobtrusive.js")" ></script>
</head>
<body>
    <div id="wrapper">
        <div class="true clearfix" id="header" style="border-bottom: 1px solid #C9DDF2">
            <div class="container clearfix">
                <a href="http://proformaclinic.se/" id="site-logo">
                    <img height="50" src="@Url.Content("~/content/images/logo.gif")" class="github-logo-4x" alt="Proforma" />
                </a>
            </div>
            <div id="dd" class="wrapper-dropdown-3">
                <span>Language</span> @ProformaApplication.Helpers.LanguageMenuHelper.LanguageMenu()
            </div>
        </div>
        <div class="site clearfix">
            <div class="context-loader-container" id="site-container">
                <div class="login_form" id="login">
                    @{Html.EnableClientValidation();}
                    @using (Html.BeginForm("Login", "Home", FormMethod.Post))
                    {
 @Html.AntiForgeryToken()
                        <h1>
                            Login
                        </h1>

                        <div class="formbody">
                            <label for="login_field">
                                @ProformaApplication.Resources.Global.UserName
                                <br>
                                @Html.TextBoxFor(model => model.UserName, new { style = "width:21em", autofocus = "true", autocomplete = "off", @class = "text" })
                            </label>
                            <label for="password">
                                @ProformaApplication.Resources.Global.Password
                                <br>
                                @Html.PasswordFor(model => model.Password, new { style = "width:21em", autocomplete = "off", @class = "text" })
                                @Html.ActionLink("Forget Password?", "ForgotPassword", "Home")
                            </label>
                            <label class="submit_btn">
                                <input type="submit" value="@ProformaApplication.Resources.Global.Login" tabindex="3" name="commit" />
                            </label>
                        </div>
                        
                    }
                </div>
            </div>
            <div class="context-overlay">
            </div>
        </div>
        <div id="footer-push">
        </div>
        <!-- hack for sticky footer -->
    </div>
    <!-- end of wrapper - hack for sticky footer -->
    <!-- footer -->
    <div id="footer">
    </div>
    <!-- /#footer -->
    <script type="text/javascript">

        function DropDown(el) {
            this.dd = el;
            this.placeholder = this.dd.children('span');
            this.opts = this.dd.find('ul.dropdown > li > a');
            this.val = '';
            this.index = -1;
            this.initEvents();
        }
        DropDown.prototype = {
            initEvents: function () {
                var obj = this;

                obj.dd.on('click', function (event) {
                    $(this).toggleClass('active');
                    return false;
                });

                obj.opts.on('click', function (e) {
                    var opt = $(this);

                    e.stopPropagation();
                });
            },
            getValue: function () {
                return this.val;
            },
            getIndex: function () {
                return this.index;
            }
        }

        $(function () {

            var dd = new DropDown($('#dd'));

            $(document).click(function () {
                // all dropdowns
                $('.wrapper-dropdown-3').removeClass('active');
            });
            $("form").tooltip({
                show: false,
                hide: false
            });


        });

        $.validator.setDefaults({

            showErrors: function (map, list) {
                // there's probably a way to simplify this
                var focussed = document.activeElement;
                if (focussed && $(focussed).is("input, textarea")) {
                    $(this.currentForm).tooltip("close", { currentTarget: focussed }, true)
                }
                this.currentElements.removeAttr("title").removeClass("input-validation-error");
                $.each(list, function (index, error) {
                    $(error.element).attr("title", error.message).addClass("input-validation-error");
                });
                if (focussed && $(focussed).is("input, textarea")) {
                    $(this.currentForm).tooltip("open", { target: focussed });
                }
            }
        });

    </script>
</body>
</html>


 public class HomeController : Controller
    {
        private readonly IAuthenticationService AuthenticationService;

        public HomeController(IAuthenticationService _authenticationSetvice)
        {
            AuthenticationService = _authenticationSetvice;
        }
        public ActionResult Index()
        {
            ViewBag.Message = "Welcome to ASP.NET MVC!";

            return View("Login");
        }
        [HttpPost]
        public ActionResult Login(LoginViewModel model, string returnUrl)
        {
            if (ModelState.IsValid)
            {

                User user = AuthenticationService.ValidateUser(model.UserName, model.Password);


                if (user.LoginStatus == AuthenticationStatus.Success)
                {
                    Session["CurrentUser"] = user;
                    var authTicket = new FormsAuthenticationTicket(1, user.UserName, DateTime.Now,
                                                       DateTime.Now.AddMinutes(30), true, 
                                                       user.TenantId + "___" + string.Join(",", user.RoleList.Select(r => r.RoleName))
                                                       +"___"+"sv-se"
                                                       );

                    string cookieContents = FormsAuthentication.Encrypt(authTicket);
                    var cookie = new HttpCookie(FormsAuthentication.FormsCookieName, cookieContents)
                    {
                        Expires = authTicket.Expiration,
                        Path = FormsAuthentication.FormsCookiePath
                    };
                    if (HttpContext != null)
                    {
                        HttpContext.Response.Cookies.Add(cookie);
                        HttpContext.Session["TenantContextId"] = user.TenantId;
                    }

                    if (Url.IsLocalUrl(returnUrl) && returnUrl.Length > 1 && returnUrl.StartsWith("/")
                        && !returnUrl.StartsWith("//") && !returnUrl.StartsWith("/\\"))
                    {
                        return Redirect(returnUrl);
                    }
                    else
                    {
                        return RedirectBasedOnUserRole(user);
                    }
                }
                else
                {
                    ModelState.AddModelError("", "The user name or password provided is incorrect.");
                }
            }
            return View();
        }

        public ActionResult About()
        {
            return View();
        }

        private ActionResult RedirectBasedOnUserRole(User user)
        {

            return RedirectToAction("Details", "Patient");
        }
    }
    
    
    
