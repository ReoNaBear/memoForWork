# memoForWork
memo
using System.Text;
using System.Text.RegularExpressions;

public static class CssCleanerLite
{
    public static string Clean(string css)
    {
        if (string.IsNullOrWhiteSpace(css))
            return css;

        // 💥 1. 移除所有 @ 規則（含 keyframes / media / font-face）
        css = RemoveAtRules(css);

        // 🧼 2. 清掉註解
        css = Regex.Replace(css, @"/\*[\s\S]*?\*/", "");

        var sb = new StringBuilder();

        // 🧩 3. 抓每個 selector block
        var matches = Regex.Matches(css, @"([^{]+)\{([^}]*)\}");

        foreach (Match match in matches)
        {
            var selector = match.Groups[1].Value.Trim();
            var body = match.Groups[2].Value;

            var cleanedBody = CleanDeclarations(body);

            if (!string.IsNullOrWhiteSpace(cleanedBody))
            {
                sb.AppendLine($"{selector} {{ {cleanedBody} }}");
            }
        }

        return sb.ToString();
    }

    private static string RemoveAtRules(string css)
    {
        // 移除所有 @xxx {...}（支援巢狀簡單情境）
        return Regex.Replace(css, @"@[^{}]+\{(?:[^{}]*|\{[^{}]*\})*\}", "", RegexOptions.IgnoreCase);
    }

    private static string CleanDeclarations(string body)
    {
        var sb = new StringBuilder();

        var declarations = body.Split(';');

        foreach (var decl in declarations)
        {
            var parts = decl.Split(':');
            if (parts.Length != 2) continue;

            var prop = parts[0].Trim().ToLower();
            var value = parts[1].Trim();

            if (IsAllowedProperty(prop))
            {
                sb.Append($"{prop}: {value}; ");
            }
        }

        return sb.ToString().Trim();
    }

    private static bool IsAllowedProperty(string prop)
    {
        // ✅ 白名單（穩定）
        return prop switch
        {
            "color" => true,
            "background" => true,
            "background-color" => true,

            "font" => true,
            "font-size" => true,
            "font-weight" => true,
            "font-family" => true,

            "text-align" => true,
            "line-height" => true,

            "margin" => true,
            "margin-top" => true,
            "margin-bottom" => true,
            "margin-left" => true,
            "margin-right" => true,

            "padding" => true,
            "padding-top" => true,
            "padding-bottom" => true,
            "padding-left" => true,
            "padding-right" => true,

            "border" => true,
            "border-top" => true,
            "border-bottom" => true,
            "border-left" => true,
            "border-right" => true,

            "width" => true,
            "height" => true,

            "display" => true, // limited support

            _ => false
        };
    }
}

private static string ConvertPercentToPx(string prop, string value, PageOrientation orientation)
{
    var match = Regex.Match(value, @"(\d+(?:\.\d+)?)%");

    if (!match.Success)
        return value;

    double percent = double.Parse(match.Groups[1].Value);

    // 📐 根據方向決定 base
    double pageWidth = orientation == PageOrientation.Portrait ? 1024 : 1448;
    double pageHeight = orientation == PageOrientation.Portrait ? 1448 : 1024;

    double baseSize = prop switch
    {
        "width" => pageWidth,
        "left" => pageWidth,
        "right" => pageWidth,

        "height" => pageHeight,
        "top" => pageHeight,
        "bottom" => pageHeight,

        "font-size" => 16,

        _ => pageWidth
    };

    double px = baseSize * (percent / 100.0);

    return $"{px:0}px";
}

private static string CleanDeclarations(string body, PageOrientation orientation)
{
    var sb = new StringBuilder();
    var declarations = body.Split(';');

    foreach (var decl in declarations)
    {
        var parts = decl.Split(':');
        if (parts.Length != 2) continue;

        var prop = parts[0].Trim().ToLower();
        var value = parts[1].Trim();

        if (IsAllowedProperty(prop) && IsSafeValue(value))
        {
            // ⭐ % → px（吃方向）
            value = ConvertPercentToPx(prop, value, orientation);

            // ⭐ 保底：全部 % 再掃一次
            value = Regex.Replace(value, @"(\d+(?:\.\d+)?)%", m =>
            {
                double percent = double.Parse(m.Groups[1].Value);

                double baseSize = orientation == PageOrientation.Portrait ? 1024 : 1448;
                double px = baseSize * (percent / 100.0);

                return $"{px:0}px";
            });

            sb.Append($"{prop}: {value}; ");
        }
    }

    return sb.ToString().Trim();
}

public static string Clean(string css, PageOrientation orientation)
{
    css = RemoveAtRulesSafe(css);

    css = Regex.Replace(css, @"/\*[\s\S]*?\*/", "");

    var sb = new StringBuilder();
    var matches = Regex.Matches(css, @"([^{]+)\{([^}]*)\}");

    foreach (Match match in matches)
    {
        var selector = match.Groups[1].Value.Trim();
        var body = match.Groups[2].Value;

        var cleanedBody = CleanDeclarations(body, orientation);

        if (!string.IsNullOrWhiteSpace(cleanedBody))
        {
            sb.AppendLine($"{selector} {{ {cleanedBody} }}");
        }
    }

    return sb.ToString();
}




新的
public enum PageOrientation
{
    Portrait,
    Landscape
}

public static class CssCleanerUltimate
{
    const double PORTRAIT_WIDTH = 1024;
    const double PORTRAIT_HEIGHT = 1448;

    const double LANDSCAPE_WIDTH = 1448;
    const double LANDSCAPE_HEIGHT = 1024;

    const double ROOT_FONT_SIZE = 16;

    public static string Clean(string css, PageOrientation orientation)
    {
        if (string.IsNullOrWhiteSpace(css))
            return css;

        css = RemoveAtRulesSafe(css);
        css = Regex.Replace(css, @"/\*[\s\S]*?\*/", "");

        var sb = new StringBuilder();
        var matches = Regex.Matches(css, @"([^{]+)\{([^}]*)\}");

        foreach (Match match in matches)
        {
            var selector = match.Groups[1].Value.Trim();
            var body = match.Groups[2].Value;

            var cleaned = CleanDeclarations(body, orientation);

            if (!string.IsNullOrWhiteSpace(cleaned))
                sb.AppendLine($"{selector} {{ {cleaned} }}");
        }

        return sb.ToString();
    }

    // 💥 不用 Regex 巢狀，避免卡死
    private static string RemoveAtRulesSafe(string css)
    {
        var sb = new StringBuilder();
        int i = 0;

        while (i < css.Length)
        {
            if (css[i] == '@')
            {
                int start = css.IndexOf('{', i);
                if (start == -1) break;

                int depth = 1;
                int j = start + 1;

                while (j < css.Length && depth > 0)
                {
                    if (css[j] == '{') depth++;
                    else if (css[j] == '}') depth--;
                    j++;
                }

                i = j;
            }
            else
            {
                sb.Append(css[i]);
                i++;
            }
        }

        return sb.ToString();
    }

    private static string CleanDeclarations(string body, PageOrientation orientation)
    {
        var sb = new StringBuilder();
        var declarations = body.Split(';');

        foreach (var decl in declarations)
        {
            var parts = decl.Split(':');
            if (parts.Length != 2) continue;

            var prop = parts[0].Trim().ToLower();
            var value = parts[1].Trim().ToLower();

            if (!IsAllowedProperty(prop)) continue;

            value = NormalizeValue(prop, value, orientation);

            if (string.IsNullOrWhiteSpace(value)) continue;

            sb.Append($"{prop}: {value}; ");
        }

        return sb.ToString().Trim();
    }

    private static string NormalizeValue(string prop, string value, PageOrientation orientation)
    {
        if (value.Contains("gradient") || value.Contains("var("))
            return null;

        // 🔥 calc()
        if (value.Contains("calc("))
            value = EvaluateCalc(value, prop, orientation);

        // 🔥 rem → px
        value = Regex.Replace(value, @"(\d+(?:\.\d+)?)rem", m =>
        {
            double rem = double.Parse(m.Groups[1].Value);
            return $"{rem * ROOT_FONT_SIZE}px";
        });

        // 🔥 % → px
        value = Regex.Replace(value, @"(\d+(?:\.\d+)?)%", m =>
        {
            double percent = double.Parse(m.Groups[1].Value);

            double width = orientation == PageOrientation.Portrait ? PORTRAIT_WIDTH : LANDSCAPE_WIDTH;
            double height = orientation == PageOrientation.Portrait ? PORTRAIT_HEIGHT : LANDSCAPE_HEIGHT;

            double baseSize = prop switch
            {
                "height" or "top" or "bottom" => height,
                _ => width
            };

            return $"{baseSize * percent / 100:0}px";
        });

        // ❌ 過濾不支援單位
        if (value.Contains("vh") || value.Contains("vw"))
            return null;

        return value;
    }

    private static string EvaluateCalc(string value, string prop, PageOrientation orientation)
    {
        try
        {
            var inner = Regex.Match(value, @"calc\((.*?)\)").Groups[1].Value;

            // 先轉 rem / %
            inner = NormalizeValue(prop, inner, orientation);

            // 支援簡單 + -
            var tokens = Regex.Split(inner, @"(\+|\-)");

            double result = 0;
            string op = "+";

            foreach (var token in tokens)
            {
                var t = token.Trim();

                if (t == "+" || t == "-")
                {
                    op = t;
                    continue;
                }

                var num = Regex.Match(t, @"(\d+(?:\.\d+)?)").Groups[1].Value;
                if (!double.TryParse(num, out double val)) continue;

                if (op == "+") result += val;
                else result -= val;
            }

            return $"{result:0}px";
        }
        catch
        {
            return null;
        }
    }

    private static bool IsAllowedProperty(string prop)
    {
        return prop switch
        {
            "color" => true,
            "background" => true,
            "background-color" => true,

            "font" => true,
            "font-size" => true,
            "font-weight" => true,
            "font-family" => true,

            "text-align" => true,
            "line-height" => true,

            "margin" or "margin-top" or "margin-bottom" or "margin-left" or "margin-right" => true,
            "padding" or "padding-top" or "padding-bottom" or "padding-left" or "padding-right" => true,

            "border" or "border-top" or "border-bottom" or "border-left" or "border-right" => true,

            "width" or "height" => true,

            "display" => true,

            _ => false
        };
    }
}
