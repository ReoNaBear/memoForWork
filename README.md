// =========================================================================
        // ⭐ 步驟 5.5：清道夫！移除空屬性與空區塊 (解決 iText 8 NRE 報錯)
        // =========================================================================
        
        // 1. 移除沒有數值的「空屬性」 (例如: "background-color: ;" 或 "color:   ;")
        // Regex 解說: 屬性名稱 [\w-]+ 接著冒號，中間可能夾雜空白，最後直接接分號
        processedCss = Regex.Replace(processedCss, @"[\w-]+\s*:\s*;", string.Empty);

        // 2. 移除沒有意義的「空區塊」 (例如: "*, ::after {  }" 或 ".loader {}")
        // Regex 解說: 抓取大括號前方不是括號的字元，加上裡面只有空白字元的 { }
        // 為了安全起見，我們讓它跑個兩次，以防有層層脫落產生的空區塊
        for (int i = 0; i < 2; i++)
        {
            processedCss = Regex.Replace(processedCss, @"[^{}]+\{\s*\}", string.Empty);
        }

public class CssFlattener
{
    public static string Process(string rawCss)
    {
        if (string.IsNullOrWhiteSpace(rawCss)) return rawCss;

        // =========================================================================
        // 步驟 1：移除不需要的現代 @ 規則 (保留 @page，因為 PDF 可能需要)
        // =========================================================================
        // 使用平衡群組 (Balanced Matching) 確保能乾淨移除包含巢狀括號的 @media 區塊
        string atRulePattern = @"@(?!page)[^{]+\{(?>[^{}]+|(?<DEPTH>)\{|(?<-DEPTH>)\})*(?(DEPTH)(?!))\}";
        string processedCss = Regex.Replace(rawCss, atRulePattern, string.Empty);

        // =========================================================================
        // 步驟 2：精準抓取所有變數宣告 (無論在哪個區塊)
        // =========================================================================
        var varMap = new Dictionary<string, string>();
        
        // 抓取格式: --變數名: 數值; (或結尾遇到 })
        var defineRegex = new Regex(@"(?<name>--[\w-]+)\s*:\s*(?<value>[^;{}]+)(?:;|(?=\}))");

        foreach (Match m in defineRegex.Matches(processedCss))
        {
            string name = m.Groups["name"].Value.Trim();
            string value = m.Groups["value"].Value.Trim();
            
            // 存入字典。如果有多個重複定義，這裡會以後面出現的為主 (符合 CSS 覆寫特性)
            varMap[name] = value; 
        }

        // =========================================================================
        // 步驟 3：剝洋蔥！處理字典內部的變數相依性 (例如 --a: var(--b, 1);)
        // =========================================================================
        bool changed = true;
        int maxLoops = 10; // 防呆機制，避免變數互相參考導致死迴圈
        
        // 支援抓取 fallback 預設值的強大 Regex (利用平衡群組處理 rgba 內的括號)
        var replaceVarRegex = new Regex(@"var\(\s*(?<name>--[\w-]+)(?:\s*,\s*(?<fallback>(?>[^()]+|\((?<DEPTH>)|\)(?<-DEPTH>))*(?(DEPTH)(?!))))?\)");

        while (changed && maxLoops > 0)
        {
            changed = false;
            foreach (var key in varMap.Keys.ToList())
            {
                if (varMap[key].Contains("var("))
                {
                    string resolvedValue = replaceVarRegex.Replace(varMap[key], m =>
                    {
                        string targetVar = m.Groups["name"].Value.Trim();
                        string fallbackVar = m.Groups["fallback"].Success ? m.Groups["fallback"].Value.Trim() : null;

                        // 1. 字典裡有解答，換進去
                        if (varMap.ContainsKey(targetVar)) return varMap[targetVar];
                        // 2. 找不到解答，但有預設值 fallback
                        if (!string.IsNullOrEmpty(fallbackVar)) return fallbackVar;
                        // 3. 都沒有，先保留原狀交給下一圈或最終替換處理
                        return m.Value; 
                    });

                    // 如果替換後的值有改變，更新字典並標記 changed，讓下一圈繼續檢查
                    if (varMap[key] != resolvedValue)
                    {
                        varMap[key] = resolvedValue;
                        changed = true;
                    }
                }
            }
            maxLoops--;
        }

        // =========================================================================
        // 步驟 4：從 CSS 中徹底刪除變數「定義列」
        // (保護正常的屬性如 box-sizing 安全留在原處，避免 iText 解析錯誤)
        // =========================================================================
        processedCss = defineRegex.Replace(processedCss, string.Empty);

        // =========================================================================
        // 步驟 5：將剩下的 CSS 裡的 var() 替換為實際數值 (遞迴清洗)
        // =========================================================================
        changed = true;
        maxLoops = 10;
        
        while (changed && maxLoops > 0)
        {
            string nextCss = replaceVarRegex.Replace(processedCss, m =>
            {
                string targetVar = m.Groups["name"].Value.Trim();
                string fallbackVar = m.Groups["fallback"].Success ? m.Groups["fallback"].Value.Trim() : null;

                if (varMap.ContainsKey(targetVar)) return varMap[targetVar];
                if (!string.IsNullOrEmpty(fallbackVar)) return fallbackVar;
                
                // 找不到時的終極保底，避免 iText 8 噴出 Object Reference 錯誤
                return "inherit"; 
            });

            changed = (nextCss != processedCss);
            processedCss = nextCss;
            maxLoops--;
        }

        // 終極保底殺手：萬一有極度畸形的 var() 躲過了上面的 Regex，全部暴力清掉
        processedCss = Regex.Replace(processedCss, @"var\([^)]+\)", "inherit");

        // =========================================================================
        // 步驟 6：加上 PDF 強制保底樣式
        // =========================================================================
        string pdfFallbackCss = @"
            table { border-collapse: collapse !important; width: 100% !important; }
            td, th { border: 0.5pt solid black !important; padding: 4px; }
        ";

        return processedCss + pdfFallbackCss;
    }
}



// Regex 解說：
// var\(\s* : 匹配 var( 以及可能的前導空白
// (?<name>--[\w-]+)        : 抓取變數名稱
// (?:\s*,\s* : (可選區塊開始) 匹配逗號，代表有備用值
// (?<fallback>(?>[^()]+|\((?<DEPTH>)|\)(?<-DEPTH>))*(?(DEPTH)(?!))) : 完美抓取逗號後面所有的備用值，即使裡面有 rgba() 也不會出錯！
// )?                       : (可選區塊結束)
// \)                       : 匹配最後的 )
var replaceVarRegex = new Regex(@"var\(\s*(?<name>--[\w-]+)(?:\s*,\s*(?<fallback>(?>[^()]+|\((?<DEPTH>)|\)(?<-DEPTH>))*(?(DEPTH)(?!))))?\)");

processedCss = replaceVarRegex.Replace(processedCss, m =>
{
    string targetVar = m.Groups["name"].Value;
    string fallbackVar = m.Groups["fallback"].Success ? m.Groups["fallback"].Value.Trim() : null;

    // 情況 A：如果字典裡有這個變數，直接用字典裡的值
    if (varMap.ContainsKey(targetVar))
    {
        return varMap[targetVar].Replace("var(", "").Replace(")", "").Trim();
    }
    
    // 情況 B：字典裡找不到，但 CSS 有提供預設值 (fallback)
    if (!string.IsNullOrEmpty(fallbackVar))
    {
        return fallbackVar; // 完美退路：使用前端設定的預設值 (例如 1 或 #000)
    }

    // 情況 C：什麼都沒有，給一個保底值避免 iText 崩潰
    return "inherit"; 
});

public class CssFlattener
{
    public static string Process(string rawCss)
    {
        if (string.IsNullOrWhiteSpace(rawCss)) return rawCss;

        // =========================================================================
        // 步驟 1：移除不需要的現代 @ 規則 (保留 @page)
        // =========================================================================
        string atRulePattern = @"@(?!page)[^{]+\{(?>[^{}]+|(?<DEPTH>)\{|(?<-DEPTH>)\})*(?(DEPTH)(?!))\}";
        string processedCss = Regex.Replace(rawCss, atRulePattern, string.Empty);

        // =========================================================================
        // 步驟 2：精準抓取所有變數宣告 (不管它在哪個區塊)
        // =========================================================================
        var varMap = new Dictionary<string, string>();
        
        // Regex 解說：
        // (?<name>--[\w-]+)  : 抓取 --開頭的變數名
        // \s*:\s* : 匹配冒號與空白
        // (?<value>[^;{}]+)  : 抓取值，直到遇到分號 ; 或大括號 } 為止
        // (?:;|(?=\}))       : 結尾必須是分號，或者是 } (對應 CSS 最後一行可能沒寫分號的情況)
        var defineRegex = new Regex(@"(?<name>--[\w-]+)\s*:\s*(?<value>[^;{}]+)(?:;|(?=\}))");

        foreach (Match m in defineRegex.Matches(processedCss))
        {
            string name = m.Groups["name"].Value.Trim();
            string value = m.Groups["value"].Value.Trim();
            varMap[name] = value; // 存入字典
        }

        // =========================================================================
        // 步驟 3：處理變數相依性 (例如 --a: var(--b);)
        // =========================================================================
        bool changed = true;
        int loopCount = 0;
        var replaceVarRegex = new Regex(@"var\((?<name>--[\w-]+)\)");

        while (changed && loopCount < 5)
        {
            changed = false;
            foreach (var key in varMap.Keys.ToList())
            {
                if (varMap[key].Contains("var("))
                {
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
        // 步驟 4：將原始 CSS 裡的變數宣告「徹底刪除」
        // (這樣 * {} 裡面的正常變數如 box-sizing 就會完美保留下來)
        // =========================================================================
        processedCss = defineRegex.Replace(processedCss, string.Empty);

        // =========================================================================
        // 步驟 5：將剩下的 CSS 裡的 var(--xxx) 替換為實際數值
        // =========================================================================
        processedCss = replaceVarRegex.Replace(processedCss, m =>
        {
            string targetVar = m.Groups["name"].Value;
            if (varMap.ContainsKey(targetVar))
            {
                return varMap[targetVar].Replace("var(", "").Replace(")", "").Trim();
            }
            return "inherit"; // 找不到時的保底防呆
        });

        // =========================================================================
        // 步驟 6：加上 PDF 強制保底樣式
        // =========================================================================
        string pdfFallbackCss = @"
            table { border-collapse: collapse !important; width: 100% !important; }
            td, th { border: 0.5pt solid black !important; padding: 4px; }
        ";

        return processedCss + pdfFallbackCss;
    }
}

.......



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
