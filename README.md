# safe_calculator.py
# A safe expression evaluator using ast (no eval).
# Supports + - * / **, parentheses and math functions (sin, cos, sqrt, etc.)

import ast
import operator
import math

# Allowed binary operators mapping
_BINOPS = {
    ast.Add: operator.add,
    ast.Sub: operator.sub,
    ast.Mult: operator.mul,
    ast.Div: operator.truediv,
    ast.Pow: operator.pow,
    ast.Mod: operator.mod,
    ast.FloorDiv: operator.floordiv,
}

# Allowed unary operators
_UNARYOPS = {
    ast.UAdd: operator.pos,
    ast.USub: operator.neg,
}

# Allowed math names (from math module) - feel free to add more
_ALLOWED_NAMES = {name: getattr(math, name) for name in (
    "sin", "cos", "tan", "asin", "acos", "atan",
    "sinh", "cosh", "tanh", "log", "log10", "sqrt",
    "exp", "floor", "ceil", "fabs", "factorial", "degrees", "radians"
)}
# also allow constants like pi and e
_ALLOWED_NAMES.update({"pi": math.pi, "e": math.e})

class SafeEvaluator(ast.NodeVisitor):
    def visit(self, node):
        method = "visit_" + node.__class__.__name__
        visitor = getattr(self, method, self.generic_visit)
        return visitor(node)

    def visit_Module(self, node):
        # expression-only mode
        if len(node.body) != 1 or not isinstance(node.body[0], ast.Expr):
            raise ValueError("Only single expressions are allowed")
        return self.visit(node.body[0].value)

    def visit_Expr(self, node):
        return self.visit(node.value)

    def visit_BinOp(self, node):
        left = self.visit(node.left)
        right = self.visit(node.right)
        op_type = type(node.op)
        if op_type in _BINOPS:
            return _BINOPS[op_type](left, right)
        raise ValueError(f"Operator {op_type} not allowed")

    def visit_UnaryOp(self, node):
        operand = self.visit(node.operand)
        op_type = type(node.op)
        if op_type in _UNARYOPS:
            return _UNARYOPS[op_type](operand)
        raise ValueError(f"Unary operator {op_type} not allowed")

    def visit_Call(self, node):
        # only allow simple calls like sin(x)
        if not isinstance(node.func, ast.Name):
            raise ValueError("Only simple function calls allowed")
        func_name = node.func.id
        if func_name not in _ALLOWED_NAMES:
            raise ValueError(f"Function '{func_name}' not allowed")
        args = [self.visit(a) for a in node.args]
        return _ALLOWED_NAMES[func_name](*args)

    def visit_Name(self, node):
        if node.id in _ALLOWED_NAMES:
            return _ALLOWED_NAMES[node.id]
        raise ValueError(f"Name '{node.id}' is not allowed")

    def visit_Num(self, node):
        return node.n

    def visit_Constant(self, node):  # for Python 3.8+
        if isinstance(node.value, (int, float)):
            return node.value
        raise ValueError("Only numeric constants are allowed")

    def generic_visit(self, node):
        raise ValueError(f"Unsupported expression: {node.__class__.__name__}")

def evaluate_expr(expr: str):
    expr = expr.strip()
    if not expr:
        raise ValueError("Empty expression")
    parsed = ast.parse(expr, mode="exec")
    evaluator = SafeEvaluator()
    return evaluator.visit(parsed)

def repl():
    print("Safe Python Calculator — type 'quit' or 'exit' to leave")
    print("Supports + - * / ** % // parentheses and math functions.")
    print("Examples: 2+2, 3*(4+5), sqrt(16), sin(pi/2)")
    while True:
        try:
            s = input(">>> ").strip()
            if s.lower() in ("quit", "exit"):
                print("Bye!")
                break
            if not s:
                continue
            result = evaluate_expr(s)
            print(result)
        except Exception as e:
            print("Error:", e)

if __name__ == "__main__":
    repl()
