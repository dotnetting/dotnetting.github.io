---
title: "My First Post"
date: 2021-04-26T18:36:24+01:00
description: "This is a short description about my first post"
tags: ["demo"]
draft: true
---

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Curabitur non risus ut magna fringilla pretium. Sed lacinia augue a ex rhoncus, quis pellentesque eros congue. Etiam facilisis condimentum auctor. Vivamus et sagittis lorem, sit amet ultrices nunc. In hac habitasse platea dictumst. Curabitur commodo feugiat arcu, imperdiet mollis nisi efficitur sit amet. Sed porta purus a elit tincidunt varius. In ultrices congue erat, eget vulputate lorem vulputate id. Donec et odio eget ante aliquet rutrum. Etiam sit amet pretium ex. Praesent eget sapien scelerisque, fermentum libero aliquet, tincidunt neque.

Integer convallis orci sit amet urna commodo commodo. Cras eget malesuada nisi. Nam commodo dapibus pellentesque. Morbi vel sem ligula. Sed vel ultrices nisi. Ut mollis ex et condimentum sodales. Cras sed lorem facilisis, pretium leo et, facilisis libero. Morbi varius pretium neque. Nunc pulvinar sed diam sit amet vehicula. Aliquam eu mattis lorem, vel commodo nisl. Suspendisse potenti. Praesent quis neque dictum, lobortis lectus ut, efficitur ex.

```csharp
public class AcceptsAttribute : Attribute, IActionConstraint
{
    private readonly string _acceptHeaderValue;

    public AcceptsAttribute(string acceptHeaderValue)
    {
        _acceptHeaderValue = acceptHeaderValue;
    }

    public bool Accept(ActionConstraintContext context)
    {
        if (!context.RouteContext.HttpContext.Request.Headers.ContainsKey("Accept"))
        {
            return false;
        }

        var headerValue = context.RouteContext.HttpContext.Request.Headers["Accept"];
        return headerValue.Contains(_acceptHeaderValue);
    }

    public int Order => 0;
}
```

Sed posuere neque et diam viverra, vel sollicitudin orci auctor. Nam porttitor euismod metus, et molestie nulla fringilla vitae. Morbi eu cursus nisi. Maecenas vel laoreet ligula. Mauris maximus nunc quis lobortis feugiat. Suspendisse sed enim nec orci viverra facilisis ut vitae ante. In id mauris eget elit lacinia ornare sed id nisi. Nunc efficitur ex ut mi scelerisque, vel finibus augue mollis. Maecenas ac ex vitae ipsum sollicitudin porta. Mauris vulputate ornare nibh quis ornare. Quisque eget metus felis.

Vivamus sed lectus convallis, sollicitudin risus ac, accumsan leo. Cras mattis at arcu interdum lobortis. Fusce sodales condimentum massa, nec rutrum lectus semper eu. Nullam efficitur augue nec auctor placerat. Proin mattis consequat leo, tincidunt scelerisque sem scelerisque volutpat. Morbi magna turpis, tincidunt auctor commodo non, consequat sagittis ante. Cras rutrum cursus dolor, ut convallis metus commodo eu. Nunc at hendrerit elit, sed vulputate tellus. Fusce nec convallis mauris. Sed vitae ipsum eros. Donec convallis ante eu ante tincidunt, imperdiet congue orci convallis. Nulla non justo vel diam sollicitudin lobortis. Aliquam ornare erat libero, ut rutrum tortor euismod vitae. Mauris eget semper elit. Nunc urna leo, tristique vitae porttitor a, tincidunt sed sapien.

Sed ex neque, commodo ut lectus nec, gravida fringilla turpis. Praesent eu pretium nisi. Duis imperdiet enim a lorem dignissim, quis pellentesque erat blandit. Vestibulum gravida non libero non luctus. Nulla finibus ultricies arcu ac auctor. Donec porttitor tellus in semper auctor. Nullam mattis mattis ultricies. Mauris in fermentum risus. Aenean malesuada cursus est, faucibus feugiat dolor rhoncus eu. Nullam at purus ac mi vehicula ornare vitae vel elit. Sed id consequat nisl. Sed orci augue, aliquam eu maximus condimentum, convallis ac nisi. Mauris et fringilla dui. Donec auctor quam nunc, non facilisis dolor varius nec.

