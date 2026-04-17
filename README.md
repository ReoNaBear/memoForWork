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
