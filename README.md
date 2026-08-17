import re

# ==========================================
# 1. EXPANDED LEXER FOR UI & PAYMENTS
# ==========================================
TOKEN_SPECIFICATION = [
    ('NUMBER',     r'\d+(\.\d+)?'),
    ('STRING',     r'"[^"]*"|\'[^\']*\''),
    # Added keywords: sign, payment, options, stripe, google, etc.
    ('KEYWORD',    r'\b(ai|agent|model|llm|server|network|api|web|mobile|desktop|ui|app|domain|sign|payment|options|identifier)\b'),
    ('OPERATOR',   r'[@.:=,]'),
    ('IDENTIFIER', r'\b[a-zA-Z_][a-zA-Z0-9_]*\b'),
    ('SKIP',       r'[ \t\n]+'),
]

def tokenize(code):
    tok_regex = '|'.join(f'(?P<{name}>{regex})' for name, regex in TOKEN_SPECIFICATION)
    tokens = []
    for mo in re.finditer(tok_regex, code):
        kind = mo.lastgroup
        value = mo.group(kind)
        if kind == 'SKIP':
            continue
        if kind == 'STRING':
            value = value[1:-1]
        tokens.append({'type': kind, 'value': value})
    return tokens

# ==========================================
# 2. UI-AWARE COMPILER / PARSER
# ==========================================
class WebUICompiler:
    def __init__(self, tokens):
        self.tokens = tokens
        self.pos = 0

    def compile(self):
        app_name = "AI Web App"
        sign_methods = []
        pay_methods = []
        network_route = "unknown"

        while self.pos < len(self.tokens):
            tok = self.tokens[self.pos]

            # Parse App Title
            if tok['type'] == 'KEYWORD' and tok['value'] == 'ai':
                if self.pos + 2 < len(self.tokens) and self.tokens[self.pos+1]['value'] == 'app':
                    app_name = self.tokens[self.pos+2]['value'].replace('_', ' ').title()
                    self.pos += 3
                    continue

            # Parse Context Properties
            if tok['type'] == 'KEYWORD' or tok['type'] == 'IDENTIFIER':
                prop_key = tok['value']
                
                # Handle multi-word properties like "sign options" or "payment options"
                if prop_key in ['sign', 'payment', 'server'] and self.pos + 1 < len(self.tokens):
                    next_tok = self.tokens[self.pos+1]
                    if next_tok['value'] in ['options', 'identifier']:
                        prop_key = f"{prop_key}_{next_tok['value']}"
                        self.pos += 1
                
                self.pos += 1 # Advance past key
                
                # Check for assignment '='
                if self.pos < len(self.tokens) and self.tokens[self.pos]['value'] == '=':
                    self.pos += 1 # Advance past '='
                    
                    # Read values until the end of the statement line
                    values = []
                    while self.pos < len(self.tokens):
                        t = self.tokens[self.pos]
                        if t['type'] in ['KEYWORD', 'IDENTIFIER', 'OPERATOR', 'NUMBER', 'STRING']:
                            if t['value'] != ',': # Strip list formatting commas
                                values.append(t['value'])
                            self.pos += 1
                        
                        # Lookahead boundary check
                        if self.pos < len(self.tokens):
                            lookahead = self.tokens[self.pos]
                            if lookahead['type'] in ['KEYWORD', 'IDENTIFIER'] and self.tokens[self.pos-1]['type'] != 'OPERATOR':
                                break
                    
                    # Assign extracted architectural tokens to UI variables
                    if prop_key == 'sign_options':
                        sign_methods = values
                    elif prop_key == 'payment_options':
                        pay_methods = values
                    elif prop_key == 'network':
                        network_route = "".join(values)
            else:
                self.pos += 1

        return self.generate_html_template(app_name, network_route, sign_methods, pay_methods)

    def generate_html_template(self, name, network, sign_list, pay_list):
        """Generates raw HTML/CSS templates matching the user code schema structures."""
        
        # Format list variables nicely for clean front-end display names
        sign_html = "".join([f'<button class="btn btn-auth">Continue with {s.title()}</button>' for s in sign_list])
        pay_html = "".join([f'<div class="pay-card">💳 {p.replace("_", " ").title()} Secure Option</div>' for p in pay_list])

        html_out = f"""<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{name}</title>
    <style>
        :root {{ --bg: #0d1117; --card: #161b22; --accent: #58a6ff; --text: #c9d1d9; }}
        body {{ font-family: -apple-system, sans-serif; background: var(--bg); color: var(--text); padding: 40px; margin: 0; }}
        .container {{ max-width: 900px; margin: 0 auto; }}
        .header {{ border-bottom: 1px solid #21262d; padding-bottom: 20px; margin-bottom: 30px; }}
        .badge {{ background: #238636; color: white; padding: 4px 8px; border-radius: 4px; font-size: 12px; }}
        .grid {{ display: grid; grid-template-columns: 1fr 1fr; gap: 20px; }}
        .card {{ background: var(--card); border: 1px solid #30363d; border-radius: 6px; padding: 24px; }}
        h2 {{ margin-top: 0; color: #fff; font-size: 20px; border-left: 4px solid var(--accent); padding-left: 10px; }}
        .btn {{ width: 100%; padding: 12px; margin-bottom: 10px; border-radius: 6px; border: 1px solid #30363d; cursor: pointer; font-weight: bold; transition: 0.2s; }}
        .btn-auth {{ background: #21262d; color: #c9d1d9; }}
        .btn-auth:hover {{ background: #30363d; color: #fff; }}
        .pay-card {{ padding: 14px; background: #21262d; border-radius: 6px; margin-bottom: 10px; border-left: 3px solid #388bfd; }}
        .footer-bar {{ margin-top: 40px; font-size: 12px; color: #8b949e; text-align: center; }}
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>{name}</h1>
            <p>📡 Linked Core Network: <code>{network}</code> | <span class="badge">Engine Online</span></p>
        </div>
        
        <div class="grid">
            <!-- SIGN IN MODULE -->
            <div class="card">
                <h2>🔐 Authentication Gate</h2>
                <p style="font-size:14px; color:#8b949e; margin-bottom:20px;">Access your LLM server account portal.</p>
                {sign_html if sign_list else '<p>No methods configured.</p>'}
            </div>

            <!-- PAYMENT OPTIONS MODULE -->
            <div class="card">
                <h2>💰 API Network Checkout</h2>
                <p style="font-size:14px; color:#8b949e; margin-bottom:20px;">Top up execution tokens securely via network endpoints.</p>
                {pay_html if pay_list else '<p>No methods configured.</p>'}
            </div>
        </div>
        
        <div class="footer-bar">
            Generated via Custom AI DSL Compiler Core Engine
        </div>
    </div>
</body>
</html>
"""
        return html_out

# ==========================================
# 3. COMPILATION RUNNER
# ==========================================
custom_code = """
ai app custom_ai_portal :
    network = api.network.server
    ui = web.app
    sign options = google, github, email
    payment options = stripe, crypto, apple_pay
    server identifier = node.llm.main
"""

# Tokenize DSL Source
tokens = tokenize(custom_code)

# Compile Tokens directly to production UI asset string
compiler = WebUICompiler(tokens)
html_output = compiler.compile()

# Print snippet preview of the compiled app architecture asset
print("--- COMPILED WEB UI TEMPLATE SNIPPET PREVIEW ---")
print("\n".join(html_output.split("\n")[20:55])) 

# Save file out to display or deploy directly
with open("index.html", "w", encoding="utf-8") as f:
    f.write(html_output)
print("\n💾 Success! Clean HTML layout generated and compiled as 'index.html'.")

