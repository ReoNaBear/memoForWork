public class CssFlattener
{
    public static string Process(string rawCss)
    {
        if (string.IsNullOrWhiteSpace(rawCss)) return rawCss;

        // =========================================================================
        // 步驟 1：移除不需要的現代 @ 規則 (如 @media, @keyframes, @supports)
        // =========================================================================
        // Regex 解說：
        // @(?!page)   : 匹配 @ 開頭，但排除 @page (因為 PDF 轉換可能需要 @page 設定邊距)
        // [^{]+       : 匹配 @ 之後直到第一個 { 之間的所有字元 (例如 media print)
        // \{          : 匹配開頭的 {
        // (?>...)* : 這是 C# 特有的「平衡群組 (Balanced Matching)」，用來處理巢狀大括號 { { } }
        // (?(DEPTH)(?!)): 確保括號完全對稱閉合
        string atRulePattern = @"@(?!page)[^{]+\{(?>[^{}]+|(?<DEPTH>)\{|(?<-DEPTH>)\})*(?(DEPTH)(?!))\}";
        string processedCss = Regex.Replace(rawCss, atRulePattern, string.Empty);

        // =========================================================================
        // 步驟 2：將 CSS 拆分為獨立的 選擇器 { 內容 } 區塊
        // =========================================================================
        // Regex 解說：
        // (?<selector>[^{}]+) : 命名群組 selector，抓取 { 之前所有不是括號的字元
        // \{                  : 匹配 {
        // (?<content>[^{}]+)  : 命名群組 content，抓取 } 之前所有不是括號的內容
        // \}                  : 匹配 }
        var blockRegex = new Regex(@"(?<selector>[^{}]+)\{(?<content>[^{}]+)\}");
        var matches = blockRegex.Matches(processedCss);

        string resetContent = "";
        List<string> otherBlocks = new List<string>();

        foreach (Match m in matches)
        {
            string selector = m.Groups["selector"].Value.Trim();
            string content = m.Groups["content"].Value.Trim();

            // 判斷是否為全域變數宣告區塊 (*, :root, :before, :after 等)
            if (selector.Contains("*") || selector.Contains(":root") || selector.Contains(":before") || selector.Contains(":after"))
            {
                resetContent += content + ";"; // 累加變數定義
            }
            else
            {
                otherBlocks.Add($"{selector} {{{content}}}"); // 其他正常樣式先存起來
            }
        }

        // =========================================================================
        // 步驟 3：解析變數，並存入 Dictionary
        // =========================================================================
        var varMap = new Dictionary<string, string>();
        
        // Regex 解說：
        // (?<name>--[\w-]+) : 抓取以 -- 開頭的變數名稱，如 --tw-bg-opacity
        // \s*:\s* : 匹配中間的冒號與可能的空白
        // (?<value>[^;]+)   : 抓取分號前所有的字元作為變數值
        var varRegex = new Regex(@"(?<name>--[\w-]+)\s*:\s*(?<value>[^;]+)");
        
        foreach (Match m in varRegex.Matches(resetContent))
        {
            string name = m.Groups["name"].Value.Trim();
            string value = m.Groups["value"].Value.Trim();
            varMap[name] = value;
        }

        // =========================================================================
        // 步驟 3.5：處理變數相依性 (例如 --a: var(--b);)
        // =========================================================================
        bool changed = true;
        int loopCount = 0;
        int maxLoops = 5; // 設定最大迴圈次數，防止無限遞迴當機
        
        // 解析 var() 的 Regex
        var replaceVarRegex = new Regex(@"var\((?<name>--[\w-]+)\)");

        while (changed && loopCount < maxLoops)
        {
            changed = false;
            foreach (var key in varMap.Keys.ToList())
            {
                if (varMap[key].Contains("var("))
                {
                    // 把變數值裡面的 var() 替換成 Dictionary 裡已知的純數值
                    string resolvedValue = replaceVarRegex.Replace(varMap[key], m =>
                    {
                        string targetVar = m.Groups["name"].Value;
                        return varMap.ContainsKey(targetVar) ? varMap[targetVar] : m.Value;
                    });

                    if (varMap[key] != resolvedValue)
                    {
                        varMap[key] = resolvedValue;
                        changed = true;
                    }
                }
            }
            loopCount++;
        }

        // =========================================================================
        // 步驟 4：遍歷其他樣式區塊，執行 var() 替換
        // =========================================================================
        List<string> finalCssBlocks = new List<string>();

        foreach (var block in otherBlocks)
        {
            // 將 block 裡面的 var(--xxx) 替換掉
            string updatedBlock = replaceVarRegex.Replace(block, m =>
            {
                string targetVar = m.Groups["name"].Value;
                
                if (varMap.ContainsKey(targetVar))
                {
                    // 去除可能因為相依性遺留下來的多餘變數結構，並防呆處理 null
                    return varMap[targetVar].Replace("var(", "").Replace(")", "").Trim(); 
                }
                
                // 如果 Dictionary 裡找不到，為了避免 iText 8 報錯，回傳 inherit 或 transparent
                return "inherit"; 
            });

            // 移除可能殘留的 --自定義屬性列 (不讓 iText 讀到它們)
            updatedBlock = Regex.Replace(updatedBlock, @"--[\w-]+:[^;]+;", string.Empty);

            finalCssBlocks.Add(updatedBlock);
        }

        // =========================================================================
        // 步驟 5：組合最終 CSS，並加上 PDF 強制保底樣式
        // =========================================================================
        string resultCss = string.Join("\n", finalCssBlocks);

        string pdfFallbackCss = @"
            table { border-collapse: collapse !important; width: 100% !important; }
            td, th { border: 0.5pt solid black !important; padding: 4px; }
        ";

        return resultCss + pdfFallbackCss;
    }
}

/saasdadsa

public static class CssColorNormalizer
{
    public static string NormalizeColor(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            return value;

        value = value.Trim().ToLower();

        // ✅ already hex
        if (Regex.IsMatch(value, @"^#([0-9a-f]{3}|[0-9a-f]{6})$"))
        {
            return ExpandHex(value);
        }

        // ✅ rgb()
        var rgb = Regex.Match(value, @"rgb\((\d+),\s*(\d+),\s*(\d+)\)");
        if (rgb.Success)
        {
            return ToHex(
                int.Parse(rgb.Groups[1].Value),
                int.Parse(rgb.Groups[2].Value),
                int.Parse(rgb.Groups[3].Value)
            );
        }

        // ✅ rgba() → ignore alpha
        var rgba = Regex.Match(value, @"rgba\((\d+),\s*(\d+),\s*(\d+),\s*[\d\.]+\)");
        if (rgba.Success)
        {
            return ToHex(
                int.Parse(rgba.Groups[1].Value),
                int.Parse(rgba.Groups[2].Value),
                int.Parse(rgba.Groups[3].Value)
            );
        }

        // ✅ named colors (基本版)
        return value switch
        {
            "black" => "#000000",
            "white" => "#ffffff",
            "red" => "#ff0000",
            "green" => "#00ff00",
            "blue" => "#0000ff",
            "gray" => "#808080",
            "grey" => "#808080",
            _ => value
        };
    }

    private static string ToHex(int r, int g, int b)
    {
        r = Clamp(r);
        g = Clamp(g);
        b = Clamp(b);

        return $"#{r:X2}{g:X2}{b:X2}".ToLower();
    }

    private static int Clamp(int v)
    {
        return v < 0 ? 0 : v > 255 ? 255 : v;
    }

    private static string ExpandHex(string hex)
    {
        if (hex.Length == 4)
        {
            // #abc → #aabbcc
            return $"#{hex[1]}{hex[1]}{hex[2]}{hex[2]}{hex[3]}{hex[3]}";
        }

        return hex;
    }
}


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
