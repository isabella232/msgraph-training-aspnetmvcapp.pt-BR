<!-- markdownlint-disable MD002 MD041 -->

Abra o Visual Studio e selecione **arquivo _GT_ novo projeto de >**. Na caixa de diálogo **novo projeto** , faça o seguinte:

1. Selecione **modelos _GT_ Visual C# _GT_ Web**.
1. Selecione **ASP.NET Web Application (.NET Framework)**.
1. Insira o **gráfico-tutorial** para o nome do projeto.

![Visual Studio 2017 criar nova caixa de diálogo de projeto](./images/vs-new-project-01.png)

> [!NOTE]
> Certifique-se de inserir exatamente o mesmo nome para o projeto do Visual Studio especificado nas instruções do laboratório. O nome do projeto do Visual Studio se torna parte do namespace no código. O código dentro dessas instruções depende do namespace correspondente ao nome do projeto do Visual Studio especificado nessas instruções. Se você usar um nome de projeto diferente, o código não será compilado, a menos que você ajuste todos os namespaces para corresponder ao nome do projeto do Visual Studio que você digitou ao criar o projeto.

Selecione **OK**. Na caixa de diálogo **novo projeto de aplicativo Web do ASP.net** , selecione **MVC** (em **ASP.NET 4.7.2 templates**) e clique em **OK**.

Pressione **F5** ou selecione **debug > iniciar depuração**. Se tudo estiver funcionando, seu navegador padrão deverá abrir e exibir uma página padrão do ASP.NET.

Antes de prosseguir, atualize o `bootstrap` pacote NuGet e instale alguns pacotes NuGet adicionais que você usará posteriormente.

- [Microsoft. Owin. host. SystemWeb](https://www.nuget.org/packages/Microsoft.Owin.Host.SystemWeb/) para habilitar as interfaces do [Owin](http://owin.org/) no aplicativo ASP.net.
- [Microsoft. Owin. Security. OpenIdConnect](https://www.nuget.org/packages/Microsoft.Owin.Security.OpenIdConnect/) para fazer a Autenticação OpenID Connect com o Azure.
- [Microsoft. Owin. Security. cookies](https://www.nuget.org/packages/Microsoft.Owin.Security.Cookies/) para habilitar a autenticação baseada em cookies.
- [Microsoft. Identity. Client](https://www.nuget.org/packages/Microsoft.Identity.Client/) para solicitar e gerenciar tokens de acesso.
- [Microsoft. Graph](https://www.nuget.org/packages/Microsoft.Graph/) para fazer chamadas para o Microsoft Graph.

Selecione **ferramentas _GT_ Gerenciador de pacotes do NuGet _GT_ console do Gerenciador de pacotes**. No console do Gerenciador de pacotes, insira os seguintes comandos.

```Powershell
Update-Package bootstrap
Install-Package Microsoft.Owin.Host.SystemWeb
Install-Package Microsoft.Owin.Security.OpenIdConnect
Install-Package Microsoft.Owin.Security.Cookies
Install-Package Microsoft.Identity.Client -Version 2.1.0-preview
Install-Package Microsoft.Graph -Version 1.11.0
```

Crie uma classe de inicialização OWIN básica. Clique com o botão `graph-tutorial` direito do mouse na pasta no Gerenciador de soluções e escolha **Adicionar > novo item**. Escolha o modelo de **classe de inicialização OWIN** , nomeie `Startup.cs`o arquivo e escolha **Adicionar**.

## <a name="design-the-app"></a>Projetar o aplicativo

Comece criando um modelo simples para uma mensagem de erro. Você usará esse modelo para mensagens de erro do flash nos modos de exibição do aplicativo.

Clique com o botão direito do mouse na pasta **modelos** no Gerenciador de soluções e escolha **Adicionar classe >...**. Nomeie a classe `Alert` e escolha **Adicionar**. Adicione o código a seguir `Alert.cs`no.

```cs
namespace graph_tutorial.Models
{
    public class Alert
    {
        public const string AlertKey = "TempDataAlerts";
        public string Message { get; set; }
        public string Debug { get; set; }
    }
}
```

Agora, atualize o layout global do aplicativo. Abra o `./Views/Shared/_Layout.cshtml` arquivo e substitua todo o conteúdo pelo código a seguir.

```html
@{
    var alerts = TempData.ContainsKey(graph_tutorial.Models.Alert.AlertKey) ?
        (List<graph_tutorial.Models.Alert>)TempData[graph_tutorial.Models.Alert.AlertKey] :
        new List<graph_tutorial.Models.Alert>();
}

<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ASP.NET Graph Tutorial</title>
    @Styles.Render("~/Content/css")
    @Scripts.Render("~/bundles/modernizr")

    <link rel="stylesheet" href="https://use.fontawesome.com/releases/v5.1.0/css/all.css"
          integrity="sha384-lKuwvrZot6UHsBSfcMvOkWwlCMgc0TaWr+30HWe3a4ltaBwTZhyTEggF5tJv8tbt"
          crossorigin="anonymous">
</head>

<body>
    <nav class="navbar navbar-expand-md navbar-dark fixed-top bg-dark">
        <div class="container">
            @Html.ActionLink("ASP.NET Graph Tutorial", "Index", "Home", new { area = "" }, new { @class = "navbar-brand" })
            <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarCollapse"
                    aria-controls="navbarCollapse" aria-expanded="false" aria-label="Toggle navigation">
                <span class="navbar-toggler-icon"></span>
            </button>
            <div class="collapse navbar-collapse" id="navbarCollapse">
                <ul class="navbar-nav mr-auto">
                    <li class="nav-item">
                        @Html.ActionLink("Home", "Index", "Home", new { area = "" },
                            new { @class = ViewBag.Current == "Home" ? "nav-link active" : "nav-link" })
                    </li>
                    @if (Request.IsAuthenticated)
                    {
                        <li class="nav-item" data-turbolinks="false">
                            @Html.ActionLink("Calendar", "Index", "Calendar", new { area = "" },
                                new { @class = ViewBag.Current == "Calendar" ? "nav-link active" : "nav-link" })
                        </li>
                    }
                </ul>
                <ul class="navbar-nav justify-content-end">
                    <li class="nav-item">
                        <a class="nav-link" href="https://developer.microsoft.com/graph/docs/concepts/overview" target="_blank">
                            <i class="fas fa-external-link-alt mr-1"></i>Docs
                        </a>
                    </li>
                    @if (Request.IsAuthenticated)
                    {
                        <li class="nav-item dropdown">
                            <a class="nav-link dropdown-toggle" data-toggle="dropdown" href="#" role="button" aria-haspopup="true" aria-expanded="false">
                                @if (!string.IsNullOrEmpty(ViewBag.User.Avatar))
                                {
                                    <img src="@ViewBag.User.Avatar" class="rounded-circle align-self-center mr-2" style="width: 32px;">
                                }
                                else
                                {
                                    <i class="far fa-user-circle fa-lg rounded-circle align-self-center mr-2" style="width: 32px;"></i>
                                }
                            </a>
                            <div class="dropdown-menu dropdown-menu-right">
                                <h5 class="dropdown-item-text mb-0">@ViewBag.User.DisplayName</h5>
                                <p class="dropdown-item-text text-muted mb-0">@ViewBag.User.Email</p>
                                <div class="dropdown-divider"></div>
                                @Html.ActionLink("Sign Out", "SignOut", "Account", new { area = "" }, new { @class = "dropdown-item" })
                            </div>
                        </li>
                    }
                    else
                    {
                        <li class="nav-item">
                            @Html.ActionLink("Sign In", "SignIn", "Account", new { area = "" }, new { @class = "nav-link" })
                        </li>
                    }
                </ul>
            </div>
        </div>
    </nav>
    <main role="main" class="container">
        @foreach (var alert in alerts)
        {
            <div class="alert alert-danger" role="alert">
                <p class="mb-3">@alert.Message</p>
                @if (!string.IsNullOrEmpty(alert.Debug))
                {
                    <pre class="alert-pre border bg-light p-2"><code>@alert.Debug</code></pre>
                }
            </div>
        }

        @RenderBody()
    </main>
    @Scripts.Render("~/bundles/jquery")
    @Scripts.Render("~/bundles/bootstrap")
    @RenderSection("scripts", required: false)
</body>
</html>
```

Este código adiciona a [inicialização](https://getbootstrap.com/) para estilos simples e a [fonte incrível](https://fontawesome.com/) para alguns ícones simples. Também define um layout global com uma barra de navegação e usa a `Alert` classe para exibir alertas.

Agora, `Content/Site.css` abra e substitua todo o conteúdo pelo código a seguir.

```css
body {
  padding-top: 4.5rem;
}

.alert-pre {
  word-wrap: break-word;
  word-break: break-all;
  white-space: pre-wrap;
}
```

Agora, atualize a página padrão. Abra o `Views/Home/index.cshtml` arquivo e substitua seu conteúdo pelo seguinte.

```html
@{
    ViewBag.Current = "Home";
}

<div class="jumbotron">
    <h1>ASP.NET Graph Tutorial</h1>
    <p class="lead">This sample app shows how to use the Microsoft Graph API to access Outlook and OneDrive data from ASP.NET</p>
    @if (Request.IsAuthenticated)
    {
        <h4>Welcome @ViewBag.User.DisplayName!</h4>
        <p>Use the navigation bar at the top of the page to get started.</p>
    }
    else
    {
        @Html.ActionLink("Click here to sign in", "SignIn", "Account", new { area = "" }, new { @class = "btn btn-primary btn-large" })
    }
</div>
```

Agora, adicione uma função auxiliar para criar `Alert` um e passá-lo para o modo de exibição. Para torná-la facilmente disponível para qualquer controlador criado, defina uma classe de controlador de base.

Clique com o botão direito do mouse na pasta **controladores** no Gerenciador de soluções e escolha **Adicionar controlador de >..**.. Escolha **controlador MVC 5-vazio** e escolha **Adicionar**. Nomeie o controlador `BaseController` e escolha **Adicionar**. Substitua o conteúdo do `BaseController.cs` pelo código a seguir.

```cs
using graph_tutorial.Models;
using System.Collections.Generic;
using System.Web.Mvc;

namespace graph_tutorial.Controllers
{
    public abstract class BaseController : Controller
    {
        protected void Flash(string message, string debug=null)
        {
            var alerts = TempData.ContainsKey(Alert.AlertKey) ?
                (List<Alert>)TempData[Alert.AlertKey] :
                new List<Alert>();

            alerts.Add(new Alert
            {
                Message = message,
                Debug = debug
            });

            TempData[Alert.AlertKey] = alerts;
        }
    }
}
```

Qualquer controlador pode herdar dessa classe de controladora base para obter acesso `Flash` à função. Atualizar a `HomeController` classe da `BaseController`qual herdar. Abra `Controllers/HomeController.cs` e altere a `public class HomeController : Controller` linha para:

```cs
public class HomeController : BaseController
```

Salve todas as suas alterações e reinicie o servidor. Agora, o aplicativo deve ser muito diferente.

![Uma captura de tela da página inicial reprojetada](./images/create-app-01.png)