"""
Manim Live Animation Editor v4.2 (PyQt6)
3-Layer OpenGL Safety + Scale/Width/Height + Fixed Position Saving

Fixes from v4.1:
- Position save now correctly updates .move_to() calls directly on the object
  instead of modifying shared position variables that other code depends on
- Shared vs private position variable detection
- Better fallback chain for position updates
- Improved parsing for objects created with chained .move_to(var)

Requirements: pip install PyQt6
"""

import sys, os, re, subprocess, threading, queue, tempfile, ast, time, textwrap
from pathlib import Path
from PyQt6.QtWidgets import (
    QApplication, QMainWindow, QWidget, QVBoxLayout, QHBoxLayout,
    QTextEdit, QPushButton, QLabel, QSplitter, QGroupBox, QGridLayout,
    QMessageBox, QFileDialog, QSpinBox, QDoubleSpinBox,
    QStatusBar, QLineEdit, QFrame, QScrollArea, QColorDialog, QCheckBox
)
from PyQt6.QtCore import Qt, QThread, pyqtSignal, QTimer
from PyQt6.QtGui import QFont, QColor, QSyntaxHighlighter, QTextCharFormat, QPalette


# ─── Syntax Highlighter ───────────────────────────────────────────────

class PythonHighlighter(QSyntaxHighlighter):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.rules = []
        kw = QTextCharFormat()
        kw.setForeground(QColor("#CC7832"))
        kw.setFontWeight(QFont.Weight.Bold)
        for w in ['class','def','return','if','else','elif','for','while',
                   'import','from','as','self','True','False','None','with',
                   'try','except','in','not','and','or','lambda','yield',
                   'pass','break','continue','raise','finally','del','global']:
            self.rules.append((re.compile(rf'\b{w}\b'), kw))
        mf = QTextCharFormat()
        mf.setForeground(QColor("#6897BB"))
        for c in ['Scene','Square','Circle','Triangle','Rectangle','Line','Arrow',
                   'Text','MathTex','Tex','Dot','VGroup','Group','NumberPlane','Axes',
                   'Create','Write','FadeIn','FadeOut','Transform','ReplacementTransform',
                   'MoveToTarget','Polygon','RegularPolygon','Star','Ellipse','Arc',
                   'Sector','RoundedRectangle','Indicate','DrawBorderThenFill',
                   'GrowFromCenter','ShrinkToCenter','Uncreate','GrowArrow',
                   'SpinInFromNothing','FadeTransform','Circumscribe','AnimationGroup',
                   'Succession','LaggedStart','Brace','SurroundingRectangle',
                   'DecimalNumber','Integer','BulletedList','Annulus','DashedLine',
                   'ParametricFunction','FunctionGraph','NumberLine','BarChart',
                   'Sphere','Cube','Cylinder','Cone','ImageMobject','SVGMobject',
                   'ThreeDScene','Surface','Arrow3D','MarkupText','Table','Matrix']:
            self.rules.append((re.compile(rf'\b{c}\b'), mf))
        sf = QTextCharFormat()
        sf.setForeground(QColor("#6A8759"))
        self.rules.append((re.compile(r'f?"[^"]*"'), sf))
        self.rules.append((re.compile(r"f?'[^']*'"), sf))
        cf = QTextCharFormat()
        cf.setForeground(QColor("#808080"))
        self.rules.append((re.compile(r'#.*'), cf))
        nf = QTextCharFormat()
        nf.setForeground(QColor("#6897BB"))
        self.rules.append((re.compile(r'\b\d+\.?\d*\b'), nf))

    def highlightBlock(self, text):
        for pat, fmt in self.rules:
            for m in pat.finditer(text):
                self.setFormat(m.start(), m.end() - m.start(), fmt)


# ═══════════════════════════════════════════════════════════════════════
#  LAYER 1: OpenGL Compatibility Preprocessor
# ═══════════════════════════════════════════════════════════════════════

class OpenGLCompatPreprocessor:

    @staticmethod
    def preprocess(code):
        warnings = []
        code, w = OpenGLCompatPreprocessor._fix_animate_set_value(code)
        warnings.extend(w)
        code, w = OpenGLCompatPreprocessor._fix_animate_set_stroke_in_loop(code)
        warnings.extend(w)
        code, w = OpenGLCompatPreprocessor._fix_replacement_transform_text(code)
        warnings.extend(w)
        code, w = OpenGLCompatPreprocessor._fix_animate_become(code)
        warnings.extend(w)
        code, w = OpenGLCompatPreprocessor._add_opengl_compatibility_header(code)
        warnings.extend(w)
        return code, warnings

    @staticmethod
    def _fix_animate_set_value(code):
        warnings = []
        lines = code.split('\n')
        new_lines = []
        i = 0
        while i < len(lines):
            line = lines[i]
            if 'self.play(' in line:
                play_lines = [line]
                paren_count = line.count('(') - line.count(')')
                j = i + 1
                while paren_count > 0 and j < len(lines):
                    play_lines.append(lines[j])
                    paren_count += lines[j].count('(') - lines[j].count(')')
                    j += 1
                play_block = '\n'.join(play_lines)
                indent = len(line) - len(line.lstrip())
                ind = ' ' * indent
                if '.animate.set_value(' in play_block:
                    fixed_block, extracted, had_fix = OpenGLCompatPreprocessor._extract_set_value_anims(
                        play_block, indent)
                    if had_fix:
                        for ext in extracted:
                            new_lines.append(f"{ind}{ext}")
                        for fl in fixed_block.split('\n'):
                            new_lines.append(fl)
                        warnings.append("Layer1: Extracted .animate.set_value() to instant apply")
                        i = j
                        continue
                for pl in play_lines:
                    new_lines.append(pl)
                i = j
            else:
                new_lines.append(line)
                i += 1
        return '\n'.join(new_lines), warnings

    @staticmethod
    def _extract_set_value_anims(play_block, indent):
        extracted = []
        had_fix = False
        sv_pattern = re.compile(r'(\w+)\.animate\.set_value\(([^)]+)\)')
        matches = list(sv_pattern.finditer(play_block))
        if not matches:
            return play_block, [], False
        for m in matches:
            extracted.append(f"{m.group(1)}.set_value({m.group(2)})")
            had_fix = True
        modified = play_block
        for m in reversed(matches):
            full_pattern = re.compile(r',?\s*' + re.escape(m.group(0)) + r'\s*,?')
            fm = full_pattern.search(modified)
            if fm:
                replacement = ',' if (modified[fm.start()] == ',' and
                    fm.end() < len(modified) and
                    modified[fm.end()-1:fm.end()] == ',') else ''
                modified = modified[:fm.start()] + replacement + modified[fm.end():]
        inner_match = re.search(r'self\.play\(\s*(.*?)\s*\)', modified, re.DOTALL)
        if inner_match:
            inner = inner_match.group(1).strip()
            cleaned = re.sub(r'\b(run_time|rate_func|lag_ratio)\s*=\s*[^,)]+,?\s*', '', inner).strip()
            cleaned = cleaned.strip(',').strip()
            if not cleaned:
                modified = re.sub(r'self\.play\([^)]*\)', 'self.wait(0.01)', modified, flags=re.DOTALL)
        return modified, extracted, had_fix

    @staticmethod
    def _fix_animate_set_stroke_in_loop(code):
        warnings = []
        if '.animate.set_stroke(' in code:
            warnings.append("Layer1: Detected .animate.set_stroke() — Layer 3 will catch if it fails")
        return code, warnings

    @staticmethod
    def _fix_replacement_transform_text(code):
        warnings = []
        text_objects = set()
        for m in re.finditer(r'(\w+)\s*=\s*(Text|MathTex|Tex|MarkupText|Integer|DecimalNumber)\s*\(', code):
            text_objects.add(m.group(1))
        def replace_rt(match):
            obj1 = match.group(1).strip()
            obj2 = match.group(2).strip()
            if obj1 in text_objects or obj2 in text_objects:
                warnings.append(f"Layer1: ReplacementTransform({obj1},{obj2}) → FadeTransform")
                return f"FadeTransform({obj1}, {obj2})"
            return match.group(0)
        code = re.sub(r'ReplacementTransform\(\s*(\w+)\s*,\s*(\w+)\s*\)', replace_rt, code)
        return code, warnings

    @staticmethod
    def _fix_animate_become(code):
        warnings = []
        if '.animate.become(' in code:
            warnings.append("Layer1: Detected .animate.become() — may need Layer 3 fallback")
        return code, warnings

    @staticmethod
    def _add_opengl_compatibility_header(code):
        warnings = []
        helper = textwrap.dedent("""\
        def _safe_apply_fallback(self, *animations, **kwargs):
            \"\"\"Fallback: apply animation targets instantly without interpolation.\"\"\"
            for anim in animations:
                if isinstance(anim, str):
                    continue
                try:
                    if hasattr(anim, 'mobject') and hasattr(anim, 'target_mobject'):
                        if anim.target_mobject is not None:
                            try:
                                anim.mobject.become(anim.target_mobject)
                            except Exception:
                                pass
                    elif hasattr(anim, 'mobject'):
                        try:
                            anim.begin()
                            anim.finish()
                            anim.clean_up_from_scene(self)
                        except Exception:
                            pass
                except Exception:
                    pass
        """)
        m = re.search(r'([ \t]*)def construct\(self\):\s*\n', code)
        if m:
            base_indent = m.group(1)
            body_indent = base_indent + '    '
            helper_lines = helper.split('\n')
            indented_helper = []
            for hl in helper_lines:
                if hl.strip():
                    indented_helper.append(body_indent + hl)
                else:
                    indented_helper.append('')
            helper_block = '\n'.join(indented_helper)
            insert_pos = m.end()
            code = code[:insert_pos] + helper_block + '\n' + code[insert_pos:]
            warnings.append("Layer1: Injected _safe_apply_fallback helper")
        return code, warnings


# ═══════════════════════════════════════════════════════════════════════
#  LAYER 3: Universal Try/Except Wrapper
# ═══════════════════════════════════════════════════════════════════════

class SafePlayInjector:

    @staticmethod
    def wrap_all_play_calls(code):
        lines = code.split('\n')
        result = []
        i = 0
        fallback_counter = 0
        while i < len(lines):
            line = lines[i]
            stripped = line.lstrip()
            if stripped.startswith('#'):
                result.append(line)
                i += 1
                continue
            if 'self.play(' in stripped and not stripped.startswith('#'):
                indent = len(line) - len(line.lstrip())
                ind = ' ' * indent
                extra_ind = ' ' * (indent + 4)
                play_lines = [line]
                paren_count = line.count('(') - line.count(')')
                j = i + 1
                while paren_count > 0 and j < len(lines):
                    play_lines.append(lines[j])
                    paren_count += lines[j].count('(') - lines[j].count(')')
                    j += 1
                play_block = '\n'.join(play_lines)
                inner_start = play_block.index('self.play(') + len('self.play(')
                depth = 1
                pos = inner_start
                for ci, ch in enumerate(play_block[inner_start:], inner_start):
                    if ch == '(': depth += 1
                    elif ch == ')':
                        depth -= 1
                        if depth == 0:
                            pos = ci
                            break
                inner_content = play_block[inner_start:pos]
                fallback_args = SafePlayInjector._extract_anim_args(inner_content)
                fallback_counter += 1
                var_name = f"_ogl_err_{fallback_counter}"
                indented_play_lines = []
                for pl in play_lines:
                    if pl.strip():
                        indented_play_lines.append(extra_ind + pl.lstrip())
                    else:
                        indented_play_lines.append(pl)
                result.append(f"{ind}try:")
                for ipl in indented_play_lines:
                    result.append(ipl)
                result.append(f"{ind}except Exception as {var_name}:")
                result.append(f"{extra_ind}print(f\"[SafePlay] Animation failed at line {i+1}: {{{var_name}}}\")")
                if fallback_args.strip():
                    result.append(f"{extra_ind}try:")
                    result.append(f"{extra_ind}    _safe_apply_fallback(self, {fallback_args})")
                    result.append(f"{extra_ind}except Exception:")
                    result.append(f"{extra_ind}    pass")
                else:
                    result.append(f"{extra_ind}pass")
                i = j
            else:
                result.append(line)
                i += 1
        return '\n'.join(result)

    @staticmethod
    def _extract_anim_args(inner_content):
        args = []
        depth = 0
        current = ''
        for ch in inner_content:
            if ch in '([{':
                depth += 1
                current += ch
            elif ch in ')]}':
                depth -= 1
                current += ch
            elif ch == ',' and depth == 0:
                args.append(current.strip())
                current = ''
            else:
                current += ch
        if current.strip():
            args.append(current.strip())
        skip_kwargs = {'run_time', 'rate_func', 'lag_ratio', 'remover',
                       'introducer', 'name', 'subcaption', 'subcaption_duration',
                       'subcaption_offset'}
        filtered = []
        for arg in args:
            kw_match = re.match(r'^(\w+)\s*=', arg)
            if kw_match and kw_match.group(1) in skip_kwargs:
                continue
            if arg.startswith('**'):
                continue
            filtered.append(arg)
        return ', '.join(filtered)


# ─── Code Processor ───────────────────────────────────────────────────

class ManimCodeProcessor:
    MANIM_OBJECTS = [
        'Square','Circle','Triangle','Rectangle','Line','Arrow','DashedLine',
        'Polygon','RegularPolygon','Star','Ellipse','Arc','Sector','Annulus',
        'AnnularSector','RoundedRectangle','Dot',
        'Text','MathTex','Tex','Integer','DecimalNumber','BulletedList','MarkupText',
        'VGroup','Group',
        'NumberPlane','Axes','NumberLine','BarChart','CoordinateSystem',
        'Sphere','Cube','Cylinder','Cone','Prism','Arrow3D','Surface',
        'SVGMobject','ImageMobject','Brace','BraceBetweenPoints',
        'ParametricFunction','FunctionGraph','SurroundingRectangle',
        'Table','Matrix','IntegerMatrix','DecimalMatrix',
    ]

    WIDTH_HEIGHT_TYPES = {'Rectangle', 'RoundedRectangle'}
    SIDE_LENGTH_TYPES = {'Square'}
    RADIUS_TYPES = {'Circle', 'Dot', 'Arc', 'Sector', 'Annulus'}
    SCALABLE_SHAPE_TYPES = (
        WIDTH_HEIGHT_TYPES | SIDE_LENGTH_TYPES | RADIUS_TYPES |
        {'Ellipse', 'RegularPolygon', 'Star', 'Triangle'}
    )
    VGROUP_TYPES = {'VGroup', 'Group'}
    ABSOLUTE_POSITION_TYPES = {
        'Polygon', 'Line', 'DashedLine', 'Arrow', 'Dot',
        'Arc', 'AnnularSector', 'Sector',
    }

    @staticmethod
    def inject_interactive_embed(code):
        lines = code.split('\n')
        new, in_multi, pc = [], False, 0
        for line in lines:
            new.append(line)
            if 'self.play(' in line and '# AUTO-INJECTED' not in line:
                pc = line.count('(') - line.count(')')
                if pc <= 0:
                    indent = len(line) - len(line.lstrip())
                    new.append(' ' * indent + "self.interactive_embed()  # AUTO-INJECTED")
                else:
                    in_multi = True
            elif in_multi:
                pc += line.count('(') - line.count(')')
                if pc <= 0:
                    in_multi = False
                    indent = len(line) - len(line.lstrip())
                    new.append(' ' * indent + "self.interactive_embed()  # AUTO-INJECTED")
        return '\n'.join(new)

    @staticmethod
    def remove_interactive_embed(code):
        return '\n'.join(l for l in code.split('\n')
                         if not ('self.interactive_embed()' in l and '# AUTO-INJECTED' in l))

    @staticmethod
    def remove_safety_wrappers(code):
        code = re.sub(
            r'\n\s*def _safe_apply_fallback\(self.*?\n(?=\s*(?:def |#|[A-Z]|\S))',
            '\n', code, flags=re.DOTALL
        )
        lines = code.split('\n')
        result = []
        i = 0
        while i < len(lines):
            line = lines[i]
            stripped = line.lstrip()
            indent = len(line) - len(stripped)
            if stripped == 'try:':
                j = i + 1
                while j < len(lines) and not lines[j].strip():
                    j += 1
                if j < len(lines) and 'self.play(' in lines[j]:
                    play_lines = []
                    k = j
                    paren_count = 0
                    while k < len(lines):
                        play_lines.append(lines[k])
                        paren_count += lines[k].count('(') - lines[k].count(')')
                        k += 1
                        if paren_count <= 0:
                            break
                    for pl in play_lines:
                        if pl.strip():
                            result.append(' ' * indent + pl.lstrip())
                        else:
                            result.append(pl)
                    while k < len(lines):
                        el = lines[k].lstrip()
                        if el.startswith('except ') and '_ogl_err' in el:
                            k += 1
                            while k < len(lines):
                                next_indent = len(lines[k]) - len(lines[k].lstrip())
                                if lines[k].strip() and next_indent <= indent:
                                    break
                                k += 1
                            break
                        elif el.strip() == '' or (len(lines[k]) - len(el) > indent):
                            k += 1
                        else:
                            break
                    i = k
                    continue
            result.append(line)
            i += 1
        return '\n'.join(result)

    @staticmethod
    def prepare_for_opengl_run(code):
        code = ManimCodeProcessor.remove_interactive_embed(code)
        code = ManimCodeProcessor.remove_safety_wrappers(code)
        code, warnings = OpenGLCompatPreprocessor.preprocess(code)
        code = SafePlayInjector.wrap_all_play_calls(code)
        code = ManimCodeProcessor._inject_embed_after_try_blocks(code)
        return code, warnings

    @staticmethod
    def _inject_embed_after_try_blocks(code):
        lines = code.split('\n')
        result = []
        i = 0
        while i < len(lines):
            line = lines[i]
            stripped = line.lstrip()
            indent = len(line) - len(stripped)
            if stripped == 'try:':
                has_play = False
                block_indent = indent
                k = i + 1
                while k < len(lines):
                    kl = lines[k].lstrip()
                    k_indent = len(lines[k]) - len(kl)
                    if 'self.play(' in lines[k]:
                        has_play = True
                    if kl.startswith('except ') and k_indent == block_indent:
                        k += 1
                        while k < len(lines):
                            if lines[k].strip() == '':
                                k += 1
                                continue
                            next_indent = len(lines[k]) - len(lines[k].lstrip())
                            if next_indent <= block_indent:
                                break
                            k += 1
                        break
                    elif kl and k_indent <= block_indent and k > i + 1 and not kl.startswith('except'):
                        break
                    k += 1
                for li in range(i, k):
                    if li < len(lines):
                        result.append(lines[li])
                if has_play:
                    result.append(' ' * block_indent + "self.interactive_embed()  # AUTO-INJECTED")
                i = k
            else:
                result.append(line)
                i += 1
        return '\n'.join(result)

    @staticmethod
    def find_matching_paren(code, start):
        pc = 0
        for i in range(start, len(code)):
            if code[i] == '(': pc += 1
            elif code[i] == ')':
                pc -= 1
                if pc == 0: return i
        return len(code) - 1

    @staticmethod
    def parse_position_variables(code):
        pos_vars = {}
        for m in re.finditer(r'(\w+)\s*=\s*np\.array\s*\(\s*\[([^\]]+)\]\s*\)', code):
            vn = m.group(1)
            try:
                coords = [float(x.strip()) for x in m.group(2).split(',')]
                coords = (coords + [0, 0, 0])[:3]
                pos_vars[vn] = coords
            except:
                pass
        for m in re.finditer(r'(\w+)\s*=\s*\[([^\]]+)\]', code):
            vn = m.group(1)
            if vn not in pos_vars:
                try:
                    coords = [float(x.strip()) for x in m.group(2).split(',')]
                    if len(coords) in (2, 3):
                        coords = (coords + [0, 0, 0])[:3]
                        pos_vars[vn] = coords
                except:
                    pass
        return pos_vars

    @staticmethod
    def _count_variable_usage(code, var_name):
        """Count how many times a variable is referenced in the code (excluding its definition)."""
        # Count all occurrences as a word boundary match
        all_uses = len(re.findall(rf'\b{re.escape(var_name)}\b', code))
        # Subtract the definition line
        has_def = 1 if re.search(rf'^[^#\n]*{re.escape(var_name)}\s*=', code, re.MULTILINE) else 0
        return all_uses - has_def

    @staticmethod
    def _parse_shape_dimensions(obj_type, params):
        dims = {
            'width': None, 'height': None,
            'side_length': None, 'radius': None,
            'original_width': None, 'original_height': None,
            'original_side_length': None, 'original_radius': None,
        }
        if obj_type in ManimCodeProcessor.WIDTH_HEIGHT_TYPES:
            wm = re.search(r'width\s*=\s*([0-9.]+)', params)
            hm = re.search(r'height\s*=\s*([0-9.]+)', params)
            dims['width'] = float(wm.group(1)) if wm else 4.0
            dims['height'] = float(hm.group(1)) if hm else 2.0
            dims['original_width'] = dims['width']
            dims['original_height'] = dims['height']
        elif obj_type in ManimCodeProcessor.SIDE_LENGTH_TYPES:
            sm = re.search(r'side_length\s*=\s*([0-9.]+)', params)
            dims['side_length'] = float(sm.group(1)) if sm else 2.0
            dims['original_side_length'] = dims['side_length']
            dims['width'] = dims['side_length']
            dims['height'] = dims['side_length']
            dims['original_width'] = dims['width']
            dims['original_height'] = dims['height']
        elif obj_type in ManimCodeProcessor.RADIUS_TYPES:
            rm = re.search(r'radius\s*=\s*([0-9.]+)', params)
            if rm:
                dims['radius'] = float(rm.group(1))
            elif obj_type == 'Circle':
                dims['radius'] = 1.0
            elif obj_type == 'Dot':
                dims['radius'] = 0.08
            dims['original_radius'] = dims['radius']
        elif obj_type == 'Ellipse':
            wm = re.search(r'width\s*=\s*([0-9.]+)', params)
            hm = re.search(r'height\s*=\s*([0-9.]+)', params)
            dims['width'] = float(wm.group(1)) if wm else 2.0
            dims['height'] = float(hm.group(1)) if hm else 1.0
            dims['original_width'] = dims['width']
            dims['original_height'] = dims['height']
        return dims

    @staticmethod
    def parse_all_objects(code):
        objects = {}
        pos_vars = ManimCodeProcessor.parse_position_variables(code)

        for obj_type in ManimCodeProcessor.MANIM_OBJECTS:
            for m in re.finditer(rf'(\w+)\s*=\s*{obj_type}\s*\(', code):
                name = m.group(1)
                ps = m.end() - 1
                pe = ManimCodeProcessor.find_matching_paren(code, ps)
                params = code[ps + 1:pe]
                has_absolute_coords = obj_type in ManimCodeProcessor.ABSOLUTE_POSITION_TYPES
                is_scalable_shape = obj_type in ManimCodeProcessor.SCALABLE_SHAPE_TYPES
                dims = ManimCodeProcessor._parse_shape_dimensions(obj_type, params)

                obj = {
                    'name': name, 'type': obj_type, 'params': params,
                    'is_text': obj_type in ['Text','MathTex','Tex','MarkupText','Integer','DecimalNumber','BulletedList'],
                    'is_group': obj_type in ['VGroup','Group'],
                    'is_scalable_shape': is_scalable_shape,
                    'has_absolute_coords': has_absolute_coords,
                    'children': [], 'all_descendants': [], 'parent': None,
                    'children_have_absolute_coords': False,
                    'position': [0, 0, 0], 'position_queried': False,
                    'position_modified': False,
                    'pos_var_name': None,
                    'pos_var_is_private': False,  # True if only this object uses the var
                    'has_chained_move_to': False,  # True if .move_to() is chained after constructor
                    'has_separate_move_to': False,  # True if obj.move_to() exists on separate line
                    'has_next_to': False,
                    'has_shift': False,
                    'font_size': None, 'original_font_size': None, 'font_modified': False,
                    'color': None, 'original_color': None, 'color_modified': False,
                    'scale': 1.0, 'scale_modified': False,
                    **dims,
                    'width_modified': False, 'height_modified': False,
                }
                fs = re.search(r'font_size\s*=\s*(\d+)', params)
                if fs:
                    obj['font_size'] = int(fs.group(1))
                    obj['original_font_size'] = int(fs.group(1))
                cm = re.search(r'color\s*=\s*(["\']?[#\w]+["\']?)', params)
                if cm:
                    c = cm.group(1).strip('"\'')
                    obj['color'] = c
                    obj['original_color'] = c

                # Detect chained calls after constructor
                after = code[pe + 1:pe + 500]

                # Walk through ALL chained calls
                rest = after
                chain_offset = pe + 1
                while rest.lstrip().startswith('.'):
                    stripped = rest.lstrip()
                    skip_ws = len(rest) - len(stripped)

                    # Check for .move_to(...)
                    mt_match = re.match(r'\.move_to\s*\(', stripped)
                    if mt_match:
                        obj['has_chained_move_to'] = True
                        paren_start = chain_offset + skip_ws + mt_match.end() - 1
                        paren_end = ManimCodeProcessor.find_matching_paren(code, paren_start)
                        mt_content = code[paren_start + 1:paren_end].strip()
                        # Check if it's a simple variable reference
                        if re.match(r'^(\w+)$', mt_content):
                            vn = mt_content
                            if vn in pos_vars:
                                obj['pos_var_name'] = vn
                                obj['position'] = list(pos_vars[vn])
                        elif re.match(r'^np\.array\s*\(\s*\[([^\]]+)\]\s*\)$', mt_content):
                            # Direct np.array — parse coords
                            cm2 = re.match(r'^np\.array\s*\(\s*\[([^\]]+)\]\s*\)$', mt_content)
                            try:
                                coords = [float(x.strip()) for x in cm2.group(1).split(',')]
                                coords = (coords + [0, 0, 0])[:3]
                                obj['position'] = coords
                            except:
                                pass
                        elif re.match(r'^\[([^\]]+)\]$', mt_content):
                            # Direct list — parse coords
                            cm3 = re.match(r'^\[([^\]]+)\]$', mt_content)
                            try:
                                coords = [float(x.strip()) for x in cm3.group(1).split(',')]
                                coords = (coords + [0, 0, 0])[:3]
                                obj['position'] = coords
                            except:
                                pass
                        chain_offset = paren_end + 1
                        rest = code[chain_offset:chain_offset + 500]
                        continue

                    # Check for .next_to(...)
                    nt_match = re.match(r'\.next_to\s*\(', stripped)
                    if nt_match:
                        obj['has_next_to'] = True
                        cp = ManimCodeProcessor.find_matching_paren(code, chain_offset + skip_ws + nt_match.end() - 1)
                        chain_offset = cp + 1
                        rest = code[chain_offset:chain_offset + 500]
                        continue

                    # Check for .shift(...)
                    sh_match = re.match(r'\.shift\s*\(', stripped)
                    if sh_match:
                        obj['has_shift'] = True
                        cp = ManimCodeProcessor.find_matching_paren(code, chain_offset + skip_ws + sh_match.end() - 1)
                        chain_offset = cp + 1
                        rest = code[chain_offset:chain_offset + 500]
                        continue

                    # Check for .scale(X)
                    sc_match = re.match(r'\.scale\s*\(\s*([0-9.]+)\s*\)', stripped)
                    if sc_match:
                        obj['scale'] = float(sc_match.group(1))
                        chain_offset += skip_ws + sc_match.end()
                        rest = code[chain_offset:chain_offset + 500]
                        continue

                    # Any other chained method — skip it
                    other_match = re.match(r'\.\w+\s*\(', stripped)
                    if other_match:
                        cp = ManimCodeProcessor.find_matching_paren(code, chain_offset + skip_ws + other_match.end() - 1)
                        chain_offset = cp + 1
                        rest = code[chain_offset:chain_offset + 500]
                    else:
                        break

                # Also check for separate obj.move_to() on its own line
                if not obj['has_chained_move_to']:
                    sep_mt = re.search(rf'(?<![.\w]){re.escape(name)}\.move_to\s*\(', code)
                    if sep_mt:
                        obj['has_separate_move_to'] = True
                        paren_start = sep_mt.end() - 1
                        paren_end = ManimCodeProcessor.find_matching_paren(code, paren_start)
                        mt_content = code[paren_start + 1:paren_end].strip()
                        if re.match(r'^(\w+)$', mt_content):
                            vn = mt_content
                            if vn in pos_vars:
                                obj['pos_var_name'] = vn
                                obj['position'] = list(pos_vars[vn])
                        elif re.match(r'^np\.array\s*\(\s*\[([^\]]+)\]\s*\)$', mt_content):
                            cm2 = re.match(r'^np\.array\s*\(\s*\[([^\]]+)\]\s*\)$', mt_content)
                            try:
                                coords = [float(x.strip()) for x in cm2.group(1).split(',')]
                                coords = (coords + [0, 0, 0])[:3]
                                obj['position'] = coords
                            except:
                                pass

                # Check separate .scale() on its own line
                if obj['scale'] == 1.0:
                    sep_sc = re.search(rf'(?<![.\w]){re.escape(name)}\.scale\s*\(\s*([0-9.]+)\s*\)', code)
                    if sep_sc:
                        obj['scale'] = float(sep_sc.group(1))

                objects[name] = obj

        # Detect which position vars are private (used by only 1 object)
        var_usage_count = {}
        for name, obj in objects.items():
            vn = obj.get('pos_var_name')
            if vn:
                if vn not in var_usage_count:
                    var_usage_count[vn] = []
                var_usage_count[vn].append(name)

        for name, obj in objects.items():
            vn = obj.get('pos_var_name')
            if vn:
                # Check if the variable is used elsewhere in the code (not just by this object)
                usage = ManimCodeProcessor._count_variable_usage(code, vn)
                obj_count = len(var_usage_count.get(vn, []))
                # Private if: only 1 object uses it AND it's referenced <=2 times total
                # (definition + one .move_to() call)
                obj['pos_var_is_private'] = (obj_count == 1 and usage <= 2)

        ManimCodeProcessor._resolve_group_children(code, objects)
        ManimCodeProcessor._detect_absolute_coord_groups(objects)
        return objects

    @staticmethod
    def _resolve_group_children(code, objects):
        for name, obj in list(objects.items()):
            if not obj['is_group']:
                continue
            children = []
            params = obj['params']
            depth, tok = 0, ''
            for ch in params + ',':
                if ch in '([{': depth += 1
                elif ch in ')]}': depth -= 1
                elif ch == ',' and depth == 0:
                    tok = tok.strip()
                    if tok and '=' not in tok and tok in objects:
                        children.append(tok)
                    tok = ''
                    continue
                tok += ch
            for am in re.finditer(rf'{re.escape(name)}\.add\s*\(([^)]*)\)', code):
                for part in am.group(1).split(','):
                    part = part.strip()
                    if part in objects and part not in children:
                        children.append(part)
            obj['children'] = list(dict.fromkeys(children))
            for cn in obj['children']:
                if cn in objects:
                    objects[cn]['parent'] = name
        for name, obj in objects.items():
            if obj['is_group']:
                obj['all_descendants'] = ManimCodeProcessor._get_all_descendants(name, objects)

    @staticmethod
    def _get_all_descendants(gn, objects):
        result = []
        obj = objects.get(gn)
        if not obj: return result
        for ch in obj.get('children', []):
            result.append(ch)
            if objects.get(ch, {}).get('is_group'):
                result.extend(ManimCodeProcessor._get_all_descendants(ch, objects))
        return result

    @staticmethod
    def _detect_absolute_coord_groups(objects):
        for name, obj in objects.items():
            if not obj['is_group']:
                continue
            has_abs = False
            for child_name in obj.get('all_descendants', []):
                child = objects.get(child_name)
                if child and child.get('has_absolute_coords'):
                    has_abs = True
                    break
            obj['children_have_absolute_coords'] = has_abs

    @staticmethod
    def parse_animations(code):
        all_objects = ManimCodeProcessor.parse_all_objects(code)
        obj_names = set(all_objects.keys())
        animations = []
        i = 0
        while i < len(code):
            m = re.search(r'self\.play\s*\(', code[i:])
            if not m: break
            start = i + m.end() - 1
            end = ManimCodeProcessor.find_matching_paren(code, start)
            content = code[start + 1:end]
            line_num = code[:start].count('\n')
            anim_objects = []
            for on in obj_names:
                if re.search(rf'\b{re.escape(on)}\b', content):
                    if on not in anim_objects:
                        anim_objects.append(on)
            expanded = list(anim_objects)
            for on in anim_objects:
                o = all_objects.get(on)
                if o and o.get('is_group'):
                    for d in o.get('all_descendants', []):
                        if d not in expanded:
                            expanded.append(d)
            anim_type = "Animation"
            for func in ['Write','FadeIn','FadeOut','Create','Transform',
                         'ReplacementTransform','DrawBorderThenFill','GrowFromCenter',
                         'Indicate','Uncreate','ShrinkToCenter','GrowArrow',
                         'SpinInFromNothing','FadeTransform','Circumscribe',
                         'LaggedStart','AnimationGroup','Succession']:
                if func in content:
                    anim_type = func; break
            if '.animate' in content:
                anim_type = "Animate"
            preview = ' '.join(content.split())
            if len(preview) > 60: preview = preview[:60] + "…"
            animations.append({
                'index': len(animations), 'line': line_num,
                'content': f"self.play({preview})", 'type': anim_type,
                'objects': expanded, 'direct_objects': anim_objects,
                'play_content': content,
            })
            i = end + 1
        return animations, all_objects

    # ═══════════════ CODE UPDATE METHODS ═══════════════

    @staticmethod
    def update_object_position(code, obj_name, new_pos, obj_data=None):
        """
        Update position for an object. Strategy priority:
        
        1. If pos_var is PRIVATE (only this object uses it):
           → Update the variable definition (safest, preserves var reference)
        
        2. If pos_var is SHARED (other code also uses it):
           → Replace .move_to(var) content directly with np.array([x,y,z])
           → Do NOT modify the shared variable
        
        3. If no pos_var but has chained .move_to():
           → Replace the content of .move_to(...)
        
        4. If has separate obj.move_to():
           → Replace the content of that call
        
        5. If has .next_to() or .shift():
           → Replace with .move_to()
        
        6. Last resort:
           → Add new obj.move_to() after definition
        """
        if obj_data:
            if (obj_data.get('is_group') and
                obj_data.get('children_have_absolute_coords') and
                not obj_data.get('pos_var_name')):
                return code

        x, y, z = [round(v, 4) for v in new_pos]
        pos_str = f"np.array([{x}, {y}, {z}])"

        pos_var = obj_data.get('pos_var_name') if obj_data else None
        pos_var_private = obj_data.get('pos_var_is_private', False) if obj_data else False

        # ─── Strategy 1: Update PRIVATE position variable ───
        if pos_var and pos_var_private:
            pat_np = rf'({re.escape(pos_var)}\s*=\s*np\.array\s*\(\s*)\[[^\]]*\](\s*\))'
            if re.search(pat_np, code):
                code = re.sub(pat_np, rf'\g<1>[{x}, {y}, {z}]\g<2>', code)
                return code
            pat_list = rf'({re.escape(pos_var)}\s*=\s*)\[[^\]]*\]'
            if re.search(pat_list, code):
                code = re.sub(pat_list, rf'\g<1>[{x}, {y}, {z}]', code)
                return code

        # ─── Strategy 2: SHARED var — replace .move_to() content directly ───
        # Find the object definition and its chained .move_to()
        obj_def = re.search(rf'{re.escape(obj_name)}\s*=\s*\w+\s*\(', code)
        if obj_def:
            ps = obj_def.end() - 1
            pe = ManimCodeProcessor.find_matching_paren(code, ps)
            after = code[pe + 1:pe + 500]

            # Walk chained calls to find .move_to()
            rest = after
            chain_offset = pe + 1
            while rest.lstrip().startswith('.'):
                stripped_rest = rest.lstrip()
                skip_ws = len(rest) - len(stripped_rest)

                mt_match = re.match(r'\.move_to\s*\(', stripped_rest)
                if mt_match:
                    paren_start = chain_offset + skip_ws + mt_match.end() - 1
                    paren_end = ManimCodeProcessor.find_matching_paren(code, paren_start)
                    # Replace content inside .move_to(...)
                    code = code[:paren_start + 1] + pos_str + code[paren_end:]
                    return code

                nt_match = re.match(r'\.next_to\s*\(', stripped_rest)
                if nt_match:
                    # Replace .next_to(...) with .move_to(pos)
                    paren_start = chain_offset + skip_ws + nt_match.end() - 1
                    paren_end = ManimCodeProcessor.find_matching_paren(code, paren_start)
                    dot_start = chain_offset + skip_ws
                    code = code[:dot_start] + f".move_to({pos_str})" + code[paren_end + 1:]
                    return code

                sh_match = re.match(r'\.shift\s*\(', stripped_rest)
                if sh_match:
                    paren_start = chain_offset + skip_ws + sh_match.end() - 1
                    paren_end = ManimCodeProcessor.find_matching_paren(code, paren_start)
                    dot_start = chain_offset + skip_ws
                    code = code[:dot_start] + f".move_to({pos_str})" + code[paren_end + 1:]
                    return code

                # Skip other chained methods
                other_match = re.match(r'\.\w+\s*\(', stripped_rest)
                if other_match:
                    cp = ManimCodeProcessor.find_matching_paren(code, chain_offset + skip_ws + other_match.end() - 1)
                    chain_offset = cp + 1
                    rest = code[chain_offset:chain_offset + 500]
                else:
                    break

        # ─── Strategy 3: Separate obj.move_to() ───
        sep = re.search(rf'(\s*){re.escape(obj_name)}\.move_to\s*\(', code)
        if sep:
            mt_s = sep.end() - 1
            mt_e = ManimCodeProcessor.find_matching_paren(code, mt_s)
            code = code[:mt_s + 1] + pos_str + code[mt_e:]
            return code

        # ─── Strategy 4: Replace separate .next_to() or .shift() ───
        sep_nt = re.search(rf'(\s*){re.escape(obj_name)}\.next_to\s*\(', code)
        if sep_nt:
            nt_s = sep_nt.end() - 1
            nt_e = ManimCodeProcessor.find_matching_paren(code, nt_s)
            indent = sep_nt.group(1)
            code = code[:sep_nt.start()] + f"{indent}{obj_name}.move_to({pos_str})" + code[nt_e + 1:]
            return code
        sep_sh = re.search(rf'(\s*){re.escape(obj_name)}\.shift\s*\(', code)
        if sep_sh:
            sh_s = sep_sh.end() - 1
            sh_e = ManimCodeProcessor.find_matching_paren(code, sh_s)
            indent = sep_sh.group(1)
            code = code[:sep_sh.start()] + f"{indent}{obj_name}.move_to({pos_str})" + code[sh_e + 1:]
            return code

        # ─── Strategy 5: Add new .move_to() after definition ───
        if obj_def and not (obj_data and obj_data.get('is_group') and obj_data.get('children_have_absolute_coords')):
            ps = obj_def.end() - 1
            pe = ManimCodeProcessor.find_matching_paren(code, ps)
            rest = code[pe + 1:]
            chain_end = pe + 1
            while rest.lstrip().startswith('.'):
                stripped = rest.lstrip()
                dot_m = re.match(r'\.\w+\s*\(', stripped)
                if dot_m:
                    skip_ws = len(rest) - len(stripped)
                    cp = ManimCodeProcessor.find_matching_paren(rest, skip_ws + dot_m.end() - 1)
                    chain_end += cp + 1
                    rest = rest[cp + 1:]
                else:
                    break
            line_end = code.find('\n', chain_end)
            if line_end == -1: line_end = len(code)
            line_start = code.rfind('\n', 0, obj_def.start()) + 1
            indent = len(code[line_start:obj_def.start()])
            code = code[:line_end] + f"\n{' ' * indent}{obj_name}.move_to({pos_str})" + code[line_end:]
        return code

    @staticmethod
    def update_object_scale(code, obj_name, scale_value):
        sv = round(scale_value, 4)
        if abs(sv - 1.0) < 0.001:
            return ManimCodeProcessor._remove_object_scale(code, obj_name)
        obj_def = re.search(rf'{re.escape(obj_name)}\s*=\s*\w+\s*\(', code)
        if obj_def:
            ps = obj_def.end() - 1
            pe = ManimCodeProcessor.find_matching_paren(code, ps)
            after = code[pe + 1:pe + 500]
            rest = after
            offset = pe + 1
            while rest.lstrip().startswith('.'):
                stripped = rest.lstrip()
                skip_ws = len(rest) - len(stripped)
                sc_match = re.match(r'\.scale\s*\(\s*[0-9.]+\s*\)', stripped)
                if sc_match:
                    abs_start = offset + skip_ws
                    abs_end = abs_start + sc_match.end()
                    code = code[:abs_start] + f".scale({sv})" + code[abs_end:]
                    return code
                dot_m = re.match(r'\.\w+\s*\(', stripped)
                if dot_m:
                    cp = ManimCodeProcessor.find_matching_paren(rest, skip_ws + dot_m.end() - 1)
                    offset += cp + 1
                    rest = rest[cp + 1:]
                else:
                    break
        sep_sc = re.search(rf'(\s*){re.escape(obj_name)}\.scale\s*\(\s*[0-9.]+\s*\)', code)
        if sep_sc:
            code = code[:sep_sc.start()] + f"{sep_sc.group(1)}{obj_name}.scale({sv})" + code[sep_sc.end():]
            return code
        if obj_def:
            ps = obj_def.end() - 1
            pe = ManimCodeProcessor.find_matching_paren(code, ps)
            rest = code[pe + 1:]
            chain_end = pe + 1
            while rest.lstrip().startswith('.'):
                stripped = rest.lstrip()
                dot_m = re.match(r'\.\w+\s*\(', stripped)
                if dot_m:
                    skip_ws = len(rest) - len(stripped)
                    cp = ManimCodeProcessor.find_matching_paren(rest, skip_ws + dot_m.end() - 1)
                    chain_end += cp + 1
                    rest = rest[cp + 1:]
                else:
                    break
            line_end = code.find('\n', chain_end)
            if line_end == -1: line_end = len(code)
            line_start = code.rfind('\n', 0, obj_def.start()) + 1
            indent = len(code[line_start:obj_def.start()])
            code = code[:line_end] + f"\n{' ' * indent}{obj_name}.scale({sv})" + code[line_end:]
        return code

    @staticmethod
    def _remove_object_scale(code, obj_name):
        obj_def = re.search(rf'{re.escape(obj_name)}\s*=\s*\w+\s*\(', code)
        if obj_def:
            ps = obj_def.end() - 1
            pe = ManimCodeProcessor.find_matching_paren(code, ps)
            after = code[pe + 1:pe + 500]
            rest = after
            offset = pe + 1
            while rest.lstrip().startswith('.'):
                stripped = rest.lstrip()
                skip_ws = len(rest) - len(stripped)
                sc_match = re.match(r'\.scale\s*\(\s*[0-9.]+\s*\)', stripped)
                if sc_match:
                    abs_start = offset + skip_ws
                    abs_end = abs_start + sc_match.end()
                    code = code[:abs_start] + code[abs_end:]
                    return code
                dot_m = re.match(r'\.\w+\s*\(', stripped)
                if dot_m:
                    cp = ManimCodeProcessor.find_matching_paren(rest, skip_ws + dot_m.end() - 1)
                    offset += cp + 1
                    rest = rest[cp + 1:]
                else:
                    break
        sep_sc = re.search(rf'\n[ \t]*{re.escape(obj_name)}\.scale\s*\(\s*[0-9.]+\s*\)\s*\n', code)
        if sep_sc:
            code = code[:sep_sc.start()] + '\n' + code[sep_sc.end():]
        return code

    @staticmethod
    def update_object_width(code, obj_name, width, obj_data=None):
        obj_type = obj_data.get('type', '') if obj_data else ''
        w = round(width, 4)
        if obj_type == 'Square':
            return ManimCodeProcessor._update_constructor_param(code, obj_name, 'side_length', w)
        elif obj_type in ('Rectangle', 'RoundedRectangle', 'Ellipse'):
            return ManimCodeProcessor._update_constructor_param(code, obj_name, 'width', w)
        elif obj_type == 'Circle':
            return ManimCodeProcessor._update_constructor_param(code, obj_name, 'radius', round(w/2, 4))
        return code

    @staticmethod
    def update_object_height(code, obj_name, height, obj_data=None):
        obj_type = obj_data.get('type', '') if obj_data else ''
        h = round(height, 4)
        if obj_type == 'Square':
            return ManimCodeProcessor._update_constructor_param(code, obj_name, 'side_length', h)
        elif obj_type in ('Rectangle', 'RoundedRectangle', 'Ellipse'):
            return ManimCodeProcessor._update_constructor_param(code, obj_name, 'height', h)
        elif obj_type == 'Circle':
            return ManimCodeProcessor._update_constructor_param(code, obj_name, 'radius', round(h/2, 4))
        return code

    @staticmethod
    def _update_constructor_param(code, obj_name, param_name, value):
        obj_def = re.search(rf'{re.escape(obj_name)}\s*=\s*\w+\s*\(', code)
        if not obj_def:
            return code
        ps = obj_def.end() - 1
        pe = ManimCodeProcessor.find_matching_paren(code, ps)
        params_str = code[ps + 1:pe]
        param_pat = re.compile(rf'({re.escape(param_name)}\s*=\s*)[0-9.]+')
        if param_pat.search(params_str):
            new_params = param_pat.sub(rf'\g<1>{value}', params_str)
            code = code[:ps + 1] + new_params + code[pe:]
        else:
            stripped = params_str.rstrip()
            if stripped:
                if not stripped.endswith(','):
                    stripped += ','
                new_params = f"{stripped} {param_name}={value}"
            else:
                new_params = f"{param_name}={value}"
            code = code[:ps + 1] + new_params + code[pe:]
        return code

    @staticmethod
    def update_object_font_size(code, obj_name, font_size):
        pat = rf'({re.escape(obj_name)}\s*=\s*\w+\s*\([^)]*font_size\s*=\s*)\d+'
        if re.search(pat, code, re.DOTALL):
            code = re.sub(pat, rf'\g<1>{font_size}', code, flags=re.DOTALL)
        return code

    @staticmethod
    def update_object_color(code, obj_name, color):
        pat = rf'({re.escape(obj_name)}\s*=\s*\w+\s*\([^)]*color\s*=\s*)["\']?[#\w]+["\']?'
        if re.search(pat, code, re.DOTALL):
            code = re.sub(pat, rf'\g<1>"{color}"', code, flags=re.DOTALL)
        return code

    @staticmethod
    def get_class_name(code):
        m = re.search(r'class\s+(\w+)\s*\([^)]*Scene[^)]*\)', code)
        return m.group(1) if m else None


# ─── Manim Runner ─────────────────────────────────────────────────────

POS_MARKER = "@@POS@@"

class ManimRunner(QThread):
    output_received = pyqtSignal(str)
    error_received = pyqtSignal(str)
    ready_for_input = pyqtSignal()
    position_received = pyqtSignal(str, list)
    process_died = pyqtSignal()
    safeplay_fallback = pyqtSignal(str)

    def __init__(self, code, class_name):
        super().__init__()
        self.code = code
        self.class_name = class_name
        self.process = None
        self.command_queue = queue.Queue()
        self.running = True
        self.temp_file = None

    def run(self):
        self.temp_file = tempfile.NamedTemporaryFile(mode='w', suffix='.py', delete=False)
        self.temp_file.write(self.code)
        self.temp_file.close()
        cmd = [sys.executable, '-m', 'manim', '-p', '--renderer=opengl',
               self.temp_file.name, self.class_name]
        try:
            self.process = subprocess.Popen(
                cmd, stdin=subprocess.PIPE, stdout=subprocess.PIPE,
                stderr=subprocess.PIPE, text=True, bufsize=1)
            def read_stdout():
                try:
                    for line in self.process.stdout:
                        line = line.strip()
                        if not line: continue
                        if POS_MARKER in line:
                            self._parse_pos(line)
                        elif '[SafePlay]' in line:
                            self.safeplay_fallback.emit(line)
                            self.output_received.emit(f"⚠️ {line}")
                        else:
                            self.output_received.emit(line)
                            if 'IPython' in line or '>>>' in line or 'embed' in line.lower():
                                self.ready_for_input.emit()
                except: pass
            def read_stderr():
                try:
                    for line in self.process.stderr:
                        line = line.strip()
                        if 'could not broadcast' in line:
                            self.error_received.emit(f"⚠️ Shape mismatch (Layer 3): {line}")
                        else:
                            self.error_received.emit(line)
                except: pass
            threading.Thread(target=read_stdout, daemon=True).start()
            threading.Thread(target=read_stderr, daemon=True).start()
            while self.running:
                try:
                    c = self.command_queue.get(timeout=0.1)
                    if c == 'STOP': break
                    if self.process and self.process.poll() is None and self.process.stdin:
                        try:
                            self.process.stdin.write(c + '\n')
                            self.process.stdin.flush()
                        except (BrokenPipeError, OSError):
                            break
                except queue.Empty:
                    if self.process and self.process.poll() is not None:
                        self.process_died.emit()
                        break
            if self.process:
                self.process.wait(timeout=5)
        except Exception as e:
            self.error_received.emit(str(e))
        finally:
            if self.temp_file and os.path.exists(self.temp_file.name):
                try: os.unlink(self.temp_file.name)
                except: pass

    def _parse_pos(self, line):
        try:
            for part in line.split(POS_MARKER):
                part = part.strip()
                if not part or '@@' not in part: continue
                segs = part.split('@@')
                if len(segs) >= 2:
                    name = segs[0].strip()
                    cs = segs[1].strip()
                    if cs:
                        coords = ast.literal_eval(cs)
                        if isinstance(coords, (list, tuple)) and len(coords) >= 2:
                            self.position_received.emit(name, [
                                float(coords[0]), float(coords[1]),
                                float(coords[2]) if len(coords) > 2 else 0.0])
        except: pass

    def send_command(self, cmd):
        self.command_queue.put(cmd)

    def query_position(self, obj_name):
        safe = obj_name.replace('"', '\\"')
        self.send_command(
            f'print("{POS_MARKER}" + "{safe}" + "@@" + str(list({obj_name}.get_center())) + "@@")')

    def is_alive(self):
        return self.process is not None and self.process.poll() is None

    def stop(self):
        self.running = False
        self.command_queue.put('STOP')
        if self.process:
            try:
                self.process.terminate()
                self.process.wait(timeout=3)
            except:
                try: self.process.kill()
                except: pass


# ─── Object Control Widget ────────────────────────────────────────────

class ObjectControlWidget(QFrame):
    position_changed = pyqtSignal(str, list)
    requery_requested = pyqtSignal(str)
    font_size_changed = pyqtSignal(str, int, float)
    color_changed = pyqtSignal(str, str)
    scale_changed = pyqtSignal(str, float)
    opacity_changed = pyqtSignal(str, float)
    width_changed = pyqtSignal(str, float)
    height_changed = pyqtSignal(str, float)

    def __init__(self, obj_data, parent=None):
        super().__init__(parent)
        self.obj_name = obj_data['name']
        self.obj_type = obj_data['type']
        self.is_text = obj_data.get('is_text', False)
        self.is_group = obj_data.get('is_group', False)
        self.is_scalable_shape = obj_data.get('is_scalable_shape', False)
        self.children = obj_data.get('children', [])
        self.all_descendants = obj_data.get('all_descendants', [])
        self.parent_group = obj_data.get('parent')
        self.pos_var_name = obj_data.get('pos_var_name')
        self.pos_var_is_private = obj_data.get('pos_var_is_private', False)
        self.children_have_absolute_coords = obj_data.get('children_have_absolute_coords', False)

        self.position = list(obj_data.get('position', [0, 0, 0]))
        self.position_queried = obj_data.get('position_queried', False)
        self.position_modified = False

        self.original_font_size = obj_data.get('original_font_size') or 24
        self.font_size = obj_data.get('font_size') or self.original_font_size
        self.font_modified = False
        self.current_font_scale = 1.0

        self.original_color = obj_data.get('original_color') or '#FFFFFF'
        self.color = obj_data.get('color') or self.original_color
        self.color_modified = False

        self.object_scale = obj_data.get('scale', 1.0)
        self.original_scale = self.object_scale
        self.scale_modified = False

        self.obj_width = obj_data.get('width')
        self.obj_height = obj_data.get('height')
        self.original_width = obj_data.get('original_width')
        self.original_height = obj_data.get('original_height')
        self.width_modified = False
        self.height_modified = False
        self.has_dimensions = (
            self.obj_type in ('Rectangle', 'RoundedRectangle', 'Square', 'Ellipse', 'Circle')
            and (self.obj_width is not None or self.obj_height is not None)
        )

        self.opacity = 1.0
        self._block = False
        self._setup_ui()

    def _setup_ui(self):
        self.setFrameStyle(QFrame.Shape.Box | QFrame.Shadow.Raised)
        bg = "#2d3d2d" if self.is_group else "#3d3d3d"
        self.setStyleSheet(f"ObjectControlWidget {{ background-color: {bg}; border-radius: 5px; margin: 2px; }}")
        layout = QVBoxLayout(self)
        layout.setContentsMargins(8, 6, 8, 6)
        layout.setSpacing(4)

        hdr = QHBoxLayout()
        icon = "📦" if self.is_group else ("📝" if self.is_text else "🔷")
        hdr.addWidget(QLabel(f"{icon} <b style='color:#64B5F6;'>{self.obj_name}</b>"))
        tc = "#81C784" if self.is_group else "#888"
        hdr.addWidget(QLabel(f"<span style='color:{tc};'>({self.obj_type})</span>"))
        if self.parent_group:
            hdr.addWidget(QLabel(f"<span style='color:#FFB74D;font-size:10px;'>∈ {self.parent_group}</span>"))
        hdr.addStretch()
        layout.addLayout(hdr)

        if self.children:
            cl = QLabel(f"  Children: {', '.join(self.children)}")
            cl.setStyleSheet("color:#888;font-size:10px;"); cl.setWordWrap(True)
            layout.addWidget(cl)
            if self.is_group:
                layout.addWidget(self._small("  ↕ Moving group moves all children", "#81C784"))
        if self.is_group and self.children_have_absolute_coords:
            layout.addWidget(self._small("  ⚠️ Children use absolute coords — group move_to skipped", "#FF9800"))
        if self.pos_var_name:
            priv = "🔒 private" if self.pos_var_is_private else "🔗 shared (direct update)"
            layout.addWidget(self._small(f"  📌 Pos var: {self.pos_var_name} ({priv})", "#64B5F6"))

        # ═══ Position ═══
        pb = QGroupBox("Position")
        pb.setStyleSheet("QGroupBox{font-size:11px;color:#aaa;}")
        pl = QVBoxLayout(pb); pl.setSpacing(3)
        self.pos_status = QLabel("⏳ Waiting for live query…")
        self.pos_status.setStyleSheet("color:#888;font-size:10px;")
        pl.addWidget(self.pos_status)
        self.pos_label = QLabel(self._pfmt())
        self.pos_label.setStyleSheet("color:#81C784;")
        pl.addWidget(self.pos_label)
        mv = QGridLayout(); mv.setSpacing(2)
        for label, r, c, dx, dy in [('↑',0,1,0,1),('↓',2,1,0,-1),('←',1,0,-1,0),('→',1,2,1,0)]:
            b = QPushButton(label); b.setFixedSize(32,32); b.setStyleSheet("font-size:14px;")
            b.clicked.connect(lambda _,dx=dx,dy=dy: self._move(dx,dy,0))
            mv.addWidget(b,r,c)
        bc = QPushButton("⊙"); bc.setFixedSize(32,32); bc.setToolTip("Move to origin")
        bc.clicked.connect(self._reset_pos); mv.addWidget(bc,1,1)
        br = QPushButton("🔄"); br.setFixedSize(32,32); br.setToolTip("Re-query position")
        br.clicked.connect(lambda: self.requery_requested.emit(self.obj_name))
        mv.addWidget(br,0,2)
        pl.addLayout(mv)
        sl = QHBoxLayout(); sl.addWidget(QLabel("Step:"))
        self.step_spin = QDoubleSpinBox()
        self.step_spin.setRange(0.1,10.0); self.step_spin.setValue(0.5)
        self.step_spin.setSingleStep(0.1); self.step_spin.setFixedWidth(60)
        sl.addWidget(self.step_spin); sl.addStretch(); pl.addLayout(sl)
        cl2 = QHBoxLayout(); cl2.addWidget(QLabel("X:"))
        self.x_spin = QDoubleSpinBox()
        self.x_spin.setRange(-20,20); self.x_spin.setDecimals(2)
        self.x_spin.setSingleStep(0.25); self.x_spin.setValue(self.position[0])
        self.x_spin.setFixedWidth(65); cl2.addWidget(self.x_spin)
        cl2.addWidget(QLabel("Y:"))
        self.y_spin = QDoubleSpinBox()
        self.y_spin.setRange(-20,20); self.y_spin.setDecimals(2)
        self.y_spin.setSingleStep(0.25); self.y_spin.setValue(self.position[1])
        self.y_spin.setFixedWidth(65); cl2.addWidget(self.y_spin)
        bg2 = QPushButton("Go"); bg2.setFixedSize(32,25)
        bg2.clicked.connect(self._go_to); cl2.addWidget(bg2)
        cl2.addStretch(); pl.addLayout(cl2)
        layout.addWidget(pb)

        # ═══ Text Properties ═══
        if self.is_text:
            tb = QGroupBox("Text Properties")
            tb.setStyleSheet("QGroupBox{font-size:11px;color:#aaa;}")
            tl = QVBoxLayout(tb); tl.setSpacing(3)
            fl = QHBoxLayout(); fl.addWidget(QLabel(f"Font (orig {self.original_font_size}):"))
            self.font_spin = QSpinBox()
            self.font_spin.setRange(8,200); self.font_spin.setValue(self.font_size)
            self.font_spin.setFixedWidth(70); self.font_spin.valueChanged.connect(self._on_font)
            fl.addWidget(self.font_spin); fl.addStretch(); tl.addLayout(fl)
            cll = QHBoxLayout(); cll.addWidget(QLabel("Color:"))
            self.color_btn = QPushButton(); self.color_btn.setFixedSize(50,22)
            self._update_color_btn(); self.color_btn.clicked.connect(self._pick_color)
            cll.addWidget(self.color_btn)
            self.color_input = QLineEdit(self.color if self.color.startswith('#') else '#FFFFFF')
            self.color_input.setFixedWidth(75); self.color_input.returnPressed.connect(self._on_color_input)
            cll.addWidget(self.color_input); cll.addStretch(); tl.addLayout(cll)
            layout.addWidget(tb)

        # ═══ Shape Dimensions ═══
        if self.has_dimensions:
            db = QGroupBox("Dimensions (saved to code)")
            db.setStyleSheet("QGroupBox{font-size:11px;color:#aaa;}")
            dl = QVBoxLayout(db); dl.setSpacing(3)
            if self.obj_type == 'Square':
                dl.addWidget(self._small(f"  Original side: {self.original_width}", "#888"))
                sl_row = QHBoxLayout(); sl_row.addWidget(QLabel("Side:"))
                self.width_spin = QDoubleSpinBox()
                self.width_spin.setRange(0.1, 50.0); self.width_spin.setValue(self.obj_width or 2.0)
                self.width_spin.setSingleStep(0.1); self.width_spin.setDecimals(2)
                self.width_spin.setFixedWidth(70)
                self.width_spin.valueChanged.connect(self._on_width)
                sl_row.addWidget(self.width_spin); sl_row.addStretch()
                dl.addLayout(sl_row)
                self.height_spin = None
            elif self.obj_type == 'Circle':
                dl.addWidget(self._small(f"  Original radius: {self.original_width and self.original_width/2}", "#888"))
                r_row = QHBoxLayout(); r_row.addWidget(QLabel("Radius:"))
                self.width_spin = QDoubleSpinBox()
                self.width_spin.setRange(0.01, 50.0)
                self.width_spin.setValue((self.obj_width or 2.0) / 2.0)
                self.width_spin.setSingleStep(0.1); self.width_spin.setDecimals(2)
                self.width_spin.setFixedWidth(70)
                self.width_spin.valueChanged.connect(self._on_circle_radius)
                r_row.addWidget(self.width_spin); r_row.addStretch()
                dl.addLayout(r_row)
                self.height_spin = None
            else:
                dl.addWidget(self._small(f"  Original: W={self.original_width}, H={self.original_height}", "#888"))
                w_row = QHBoxLayout(); w_row.addWidget(QLabel("Width:"))
                self.width_spin = QDoubleSpinBox()
                self.width_spin.setRange(0.1, 50.0); self.width_spin.setValue(self.obj_width or 4.0)
                self.width_spin.setSingleStep(0.1); self.width_spin.setDecimals(2)
                self.width_spin.setFixedWidth(70)
                self.width_spin.valueChanged.connect(self._on_width)
                w_row.addWidget(self.width_spin); w_row.addStretch()
                dl.addLayout(w_row)
                h_row = QHBoxLayout(); h_row.addWidget(QLabel("Height:"))
                self.height_spin = QDoubleSpinBox()
                self.height_spin.setRange(0.1, 50.0); self.height_spin.setValue(self.obj_height or 2.0)
                self.height_spin.setSingleStep(0.1); self.height_spin.setDecimals(2)
                self.height_spin.setFixedWidth(70)
                self.height_spin.valueChanged.connect(self._on_height)
                h_row.addWidget(self.height_spin); h_row.addStretch()
                dl.addLayout(h_row)
            layout.addWidget(db)

        # ═══ Scale & Opacity ═══
        tf = QGroupBox("Transform")
        tf.setStyleSheet("QGroupBox{font-size:11px;color:#aaa;}")
        tfl = QVBoxLayout(tf); tfl.setSpacing(3)
        scl = QHBoxLayout(); scl.addWidget(QLabel("Scale:"))
        self.scale_spin = QDoubleSpinBox()
        self.scale_spin.setRange(0.1, 10.0); self.scale_spin.setValue(self.object_scale)
        self.scale_spin.setSingleStep(0.1); self.scale_spin.setFixedWidth(65)
        self.scale_spin.valueChanged.connect(self._on_scale)
        scl.addWidget(self.scale_spin)
        self.scale_status = QLabel("✅" if abs(self.object_scale - 1.0) > 0.001 else "")
        self.scale_status.setStyleSheet("color:#81C784;font-size:9px;")
        scl.addWidget(self.scale_status); scl.addStretch()
        tfl.addLayout(scl)
        tfl.addWidget(self._small("Scale saved to code on Save & Re-run", "#81C784"))
        opl = QHBoxLayout(); opl.addWidget(QLabel("Opacity:"))
        self.opacity_spin = QDoubleSpinBox()
        self.opacity_spin.setRange(0.0, 1.0); self.opacity_spin.setValue(1.0)
        self.opacity_spin.setSingleStep(0.1); self.opacity_spin.setFixedWidth(65)
        self.opacity_spin.valueChanged.connect(self._on_opacity)
        opl.addWidget(self.opacity_spin)
        opl.addWidget(self._small("(preview only)", "#FFB74D"))
        opl.addStretch(); tfl.addLayout(opl)
        layout.addWidget(tf)

    def _small(self, txt, color):
        l = QLabel(txt); l.setStyleSheet(f"color:{color};font-size:9px;"); l.setWordWrap(True); return l

    def _pfmt(self):
        m = " ✏️" if self.position_modified else ""
        return f"📍 ({self.position[0]:.2f}, {self.position[1]:.2f}, {self.position[2]:.2f}){m}"

    def update_live_position(self, pos):
        self._block = True
        self.position = [round(v,4) for v in pos]
        self.position_queried = True
        if not self.position_modified:
            self.pos_status.setText("✅ Live position"); self.pos_status.setStyleSheet("color:#81C784;font-size:10px;")
        self.pos_label.setText(self._pfmt())
        self.x_spin.setValue(self.position[0]); self.y_spin.setValue(self.position[1])
        self._block = False

    def apply_delta(self, dx, dy, dz):
        self.position[0]+=dx; self.position[1]+=dy; self.position[2]+=dz
        self.position_modified = True; self._refresh()

    def _move(self, dx, dy, dz):
        s = self.step_spin.value()
        self.position[0]+=dx*s; self.position[1]+=dy*s; self.position[2]+=dz*s
        self.position_modified = True; self._refresh()
        self.position_changed.emit(self.obj_name, self.position.copy())

    def _reset_pos(self):
        self.position=[0,0,0]; self.position_modified=True; self._refresh()
        self.position_changed.emit(self.obj_name,[0,0,0])

    def _go_to(self):
        self.position[0]=self.x_spin.value(); self.position[1]=self.y_spin.value()
        self.position_modified=True; self._refresh()
        self.position_changed.emit(self.obj_name, self.position.copy())

    def _refresh(self):
        self.pos_label.setText(self._pfmt())
        if not self._block:
            self._block=True
            self.x_spin.setValue(self.position[0]); self.y_spin.setValue(self.position[1])
            self._block=False

    def _on_font(self, val):
        self.font_size=val; self.font_modified=True
        ns=val/self.original_font_size; ratio=ns/self.current_font_scale; self.current_font_scale=ns
        self.font_size_changed.emit(self.obj_name,val,ratio)

    def _pick_color(self):
        c = QColorDialog.getColor(QColor(self.color),self,"Select Color")
        if c.isValid():
            self.color=c.name(); self.color_modified=True
            self.color_input.setText(self.color); self._update_color_btn()
            self.color_changed.emit(self.obj_name,self.color)

    def _on_color_input(self):
        t=self.color_input.text().strip()
        if re.match(r'^#[0-9a-fA-F]{6},t):
            self.color=t; self.color_modified=True; self._update_color_btn()
            self.color_changed.emit(self.obj_name,self.color)

    def _update_color_btn(self):
        try: self.color_btn.setStyleSheet(f"background-color:{self.color};border:1px solid #666;")
        except: pass

    def _on_scale(self, val):
        old_scale = self.object_scale
        self.object_scale = val
        self.scale_modified = (abs(val - self.original_scale) > 0.001)
        self.scale_status.setText("✏️" if self.scale_modified else "")
        if old_scale > 0:
            self.scale_changed.emit(self.obj_name, val / old_scale)

    def _on_width(self, val):
        if self._block: return
        self.obj_width = val
        self.width_modified = True
        if self.obj_type == 'Square':
            self.obj_height = val
            self.height_modified = True
        self.width_changed.emit(self.obj_name, val)

    def _on_height(self, val):
        if self._block: return
        self.obj_height = val
        self.height_modified = True
        self.height_changed.emit(self.obj_name, val)

    def _on_circle_radius(self, val):
        if self._block: return
        self.obj_width = val * 2
        self.obj_height = val * 2
        self.width_modified = True
        self.height_modified = True
        self.width_changed.emit(self.obj_name, val * 2)

    def _on_opacity(self, val):
        self.opacity=val; self.opacity_changed.emit(self.obj_name,val)


# ─── Animation Card ───────────────────────────────────────────────────

class AnimationCard(QFrame):
    clicked = pyqtSignal(int)
    def __init__(self, ad, cur=False, parent=None):
        super().__init__(parent); self.idx=ad['index']; self.data=ad; self.is_current=cur; self._ui()
    def _ui(self):
        self._style(); self.setCursor(Qt.CursorShape.PointingHandCursor)
        lay=QVBoxLayout(self); lay.setContentsMargins(8,5,8,5); lay.setSpacing(2)
        h=QHBoxLayout()
        h.addWidget(QLabel(f"<b style='color:#64B5F6;'>#{self.idx+1}</b>"))
        h.addWidget(QLabel(f"<span style='color:#81C784;font-size:11px;'>[{self.data['type']}]</span>"))
        h.addStretch()
        nd=len(self.data['direct_objects']); nt=len(self.data['objects'])
        ct=f"🎯{nd}"+( f"(+{nt-nd})" if nt>nd else "")
        h.addWidget(QLabel(f"<span style='color:#FFB74D;font-size:11px;'>{ct}</span>"))
        lay.addLayout(h)
        if self.data['direct_objects']:
            t=", ".join(self.data['direct_objects'])
            if len(t)>45: t=t[:45]+"…"
            l=QLabel(t); l.setStyleSheet("color:#aaa;font-size:10px;"); l.setWordWrap(True); lay.addWidget(l)
    def _style(self):
        if self.is_current:
            self.setStyleSheet("AnimationCard{background-color:#1E3A5F;border:2px solid #2196F3;border-radius:6px;}")
        else:
            self.setStyleSheet("AnimationCard{background-color:#2d2d2d;border:1px solid #444;border-radius:6px;}AnimationCard:hover{background-color:#3d3d3d;}")
    def set_current(self, v): self.is_current=v; self._style()
    def mousePressEvent(self, e): self.clicked.emit(self.idx); super().mousePressEvent(e)


# ─── Main Editor ──────────────────────────────────────────────────────

class ManimLiveEditor(QMainWindow):
    def __init__(self):
        super().__init__()
        self.manim_runner = None
        self.all_objects = {}
        self.animations = []
        self.current_anim_idx = 0
        self.obj_widgets = {}
        self.anim_cards = []
        self.interactive_count = 0
        self.layer_warnings = []
        self._setup_ui()

    def _setup_ui(self):
        self.setWindowTitle("Manim Live Editor v4.2 (Position Fix + Scale + Dimensions)")
        self.setGeometry(80,80,1520,920)
        central=QWidget(); self.setCentralWidget(central)
        ml=QHBoxLayout(central)
        splitter=QSplitter(Qt.Orientation.Horizontal); ml.addWidget(splitter)

        # Left: Code
        left=QWidget(); ll=QVBoxLayout(left)
        h=QHBoxLayout(); h.addWidget(QLabel("<h3>📝 Code Editor</h3>"))
        bl=QPushButton("📂 Load"); bl.clicked.connect(self._load); h.addWidget(bl)
        bs=QPushButton("💾 Save"); bs.clicked.connect(self._save); h.addWidget(bs)
        h.addStretch(); ll.addLayout(h)
        self.code_editor=QTextEdit(); self.code_editor.setFont(QFont("Consolas",11))
        self.code_editor.setPlaceholderText("Paste your Manim code here…")
        self.highlighter=PythonHighlighter(self.code_editor.document())
        ll.addWidget(self.code_editor)
        btns=QHBoxLayout()
        bp=QPushButton("🔍 Parse Code"); bp.clicked.connect(self._parse)
        bp.setStyleSheet("background-color:#FF9800;color:white;font-weight:bold;padding:6px;")
        btns.addWidget(bp)
        bi=QPushButton("💉 Inject"); bi.clicked.connect(lambda: self.code_editor.setPlainText(
            ManimCodeProcessor.inject_interactive_embed(self.code_editor.toPlainText())))
        btns.addWidget(bi)
        bc=QPushButton("🧹 Clean"); bc.clicked.connect(self._clean_code)
        btns.addWidget(bc)
        ll.addLayout(btns)
        self.layer_info = QLabel("")
        self.layer_info.setStyleSheet("color:#FFB74D;font-size:10px;padding:2px;")
        self.layer_info.setWordWrap(True)
        ll.addWidget(self.layer_info)
        splitter.addWidget(left)

        # Middle
        mid=QWidget(); ml2=QVBoxLayout(mid)
        ah=QHBoxLayout(); ah.addWidget(QLabel("<h3>🎬 Animations</h3>"))
        self.anim_counter=QLabel("0/0"); self.anim_counter.setStyleSheet("color:#64B5F6;font-size:14px;font-weight:bold;")
        ah.addWidget(self.anim_counter); ah.addStretch(); ml2.addLayout(ah)
        asc=QScrollArea(); asc.setWidgetResizable(True); asc.setMaximumHeight(180)
        asc.setHorizontalScrollBarPolicy(Qt.ScrollBarPolicy.ScrollBarAlwaysOff)
        self.anim_container=QWidget()
        self.anim_layout=QVBoxLayout(self.anim_container); self.anim_layout.setSpacing(3); self.anim_layout.addStretch()
        asc.setWidget(self.anim_container); ml2.addWidget(asc)
        nav=QHBoxLayout()
        self.btn_prev=QPushButton("⏮ Prev"); self.btn_prev.clicked.connect(self._prev); self.btn_prev.setEnabled(False); nav.addWidget(self.btn_prev)
        self.btn_next=QPushButton("Next ⏭"); self.btn_next.clicked.connect(self._next_ui); self.btn_next.setEnabled(False); nav.addWidget(self.btn_next)
        ml2.addLayout(nav)
        self.cur_label=QLabel("<b>Current:</b> None")
        self.cur_label.setStyleSheet("color:#81C784;padding:4px;background-color:#2d2d2d;border-radius:4px;font-size:11px;")
        self.cur_label.setWordWrap(True); ml2.addWidget(self.cur_label)
        oh=QHBoxLayout(); oh.addWidget(QLabel("<h3>🎯 Objects</h3>"))
        self.obj_counter=QLabel("0"); self.obj_counter.setStyleSheet("color:#FFB74D;"); oh.addWidget(self.obj_counter)
        oh.addStretch()
        self.cb_all=QCheckBox("Show all"); self.cb_all.setStyleSheet("color:#888;"); self.cb_all.toggled.connect(self._refresh_obj)
        oh.addWidget(self.cb_all); ml2.addLayout(oh)
        osc=QScrollArea(); osc.setWidgetResizable(True)
        osc.setHorizontalScrollBarPolicy(Qt.ScrollBarPolicy.ScrollBarAlwaysOff)
        self.obj_container=QWidget()
        self.obj_layout=QVBoxLayout(self.obj_container); self.obj_layout.addStretch()
        osc.setWidget(self.obj_container); ml2.addWidget(osc)

        self.btn_save_rerun=QPushButton("💾 Save Changes & Re-run")
        self.btn_save_rerun.setStyleSheet(
            "background-color:#E65100;color:white;font-size:14px;font-weight:bold;padding:10px;border-radius:6px;")
        self.btn_save_rerun.clicked.connect(self._save_and_rerun)
        ml2.addWidget(self.btn_save_rerun)
        info=QLabel("🔄 Live edits are temporary previews.\n💾 Save & Re-run to apply permanently.\n🛡️ 3-Layer safety | 📐 Scale+Dims | 📌 Smart position vars")
        info.setStyleSheet("color:#FFB74D;font-size:10px;padding:4px;"); info.setWordWrap(True)
        ml2.addWidget(info)
        splitter.addWidget(mid)

        # Right
        right=QWidget(); rl=QVBoxLayout(right)
        rl.addWidget(QLabel("<h3>▶ Control</h3>"))
        cl=QHBoxLayout(); cl.addWidget(QLabel("Scene:"))
        self.class_input=QLineEdit(); self.class_input.setPlaceholderText("Auto-detected…")
        cl.addWidget(self.class_input); rl.addLayout(cl)
        rb=QHBoxLayout()
        self.btn_run=QPushButton("▶ Run"); self.btn_run.clicked.connect(self._run)
        self.btn_run.setStyleSheet("background-color:#2196F3;color:white;font-size:13px;padding:8px;font-weight:bold;")
        rb.addWidget(self.btn_run)
        self.btn_stop=QPushButton("⏹ Stop"); self.btn_stop.clicked.connect(self._stop)
        self.btn_stop.setEnabled(False); self.btn_stop.setStyleSheet("background-color:#f44336;color:white;")
        rb.addWidget(self.btn_stop); rl.addLayout(rb)

        ig=QGroupBox("Interactive Mode"); il=QVBoxLayout(ig)
        self.btn_next_manim=QPushButton("⏭ Next Animation (preview only)")
        self.btn_next_manim.clicked.connect(self._next_manim); self.btn_next_manim.setEnabled(False)
        self.btn_next_manim.setStyleSheet("background-color:#673AB7;color:white;font-weight:bold;")
        il.addWidget(self.btn_next_manim)
        self.btn_requery=QPushButton("🔄 Re-query All Positions")
        self.btn_requery.clicked.connect(self._query_all); self.btn_requery.setEnabled(False)
        il.addWidget(self.btn_requery)
        cmdl=QHBoxLayout()
        self.cmd_input=QLineEdit(); self.cmd_input.setPlaceholderText("Custom command…")
        self.cmd_input.returnPressed.connect(self._send_cmd); cmdl.addWidget(self.cmd_input)
        self.btn_send=QPushButton("Send"); self.btn_send.clicked.connect(self._send_cmd)
        self.btn_send.setEnabled(False); cmdl.addWidget(self.btn_send)
        il.addLayout(cmdl); rl.addWidget(ig)

        rl.addWidget(QLabel("<h4>Console</h4>"))
        self.console=QTextEdit(); self.console.setReadOnly(True)
        self.console.setFont(QFont("Consolas",9)); self.console.setStyleSheet("background-color:#1e1e1e;color:#dcdcdc;")
        rl.addWidget(self.console)
        eb=QHBoxLayout()
        be=QPushButton("📤 Export Clean"); be.clicked.connect(self._export)
        be.setStyleSheet("background-color:#9C27B0;color:white;"); eb.addWidget(be)
        ba=QPushButton("📝 Save to Code Only"); ba.clicked.connect(self._apply_to_code)
        ba.setStyleSheet("background-color:#4CAF50;color:white;"); eb.addWidget(ba)
        rl.addLayout(eb)
        splitter.addWidget(right)
        splitter.setSizes([460,450,370])
        self.status_bar=QStatusBar(); self.setStatusBar(self.status_bar)
        self.status_bar.showMessage("Ready — v4.2 Smart Position + Scale + Dimensions")

    def _clean_code(self):
        code = self.code_editor.toPlainText()
        code = ManimCodeProcessor.remove_interactive_embed(code)
        code = ManimCodeProcessor.remove_safety_wrappers(code)
        self.code_editor.setPlainText(code)
        self._log("🧹 Cleaned")

    def _parse(self):
        code=self.code_editor.toPlainText()
        self.animations, self.all_objects = ManimCodeProcessor.parse_animations(code)
        for c in self.anim_cards: c.deleteLater()
        self.anim_cards.clear()
        while self.anim_layout.count():
            it=self.anim_layout.takeAt(0)
            if it.widget(): it.widget().deleteLater()
        for a in self.animations:
            card=AnimationCard(a, cur=(a['index']==0))
            card.clicked.connect(self._on_anim_click); self.anim_layout.addWidget(card); self.anim_cards.append(card)
        self.anim_layout.addStretch()
        self.current_anim_idx=0; self.interactive_count=0
        self._refresh_anim()
        cn=ManimCodeProcessor.get_class_name(code)
        if cn: self.class_input.setText(cn)
        na,no=len(self.animations),len(self.all_objects)
        self.status_bar.showMessage(f"Parsed: {na} anims, {no} objects")
        self._log(f"✓ {na} animations, {no} objects")
        for n,o in self.all_objects.items():
            if o.get('pos_var_name'):
                priv = "private" if o.get('pos_var_is_private') else "SHARED"
                self._log(f"  📌 {n} → {o['pos_var_name']} ({priv})")
            if o.get('has_chained_move_to'):
                self._log(f"  🔗 {n}: chained .move_to()")
            if o['is_group'] and o['children']:
                self._log(f"  📦 {n}: [{', '.join(o['children'])}]")

    def _on_anim_click(self, idx):
        self.current_anim_idx=idx; self._refresh_anim()

    def _refresh_anim(self):
        for i,c in enumerate(self.anim_cards): c.set_current(i==self.current_anim_idx)
        t=len(self.animations)
        self.anim_counter.setText(f"{self.current_anim_idx+1}/{t}" if t else "0/0")
        self.btn_prev.setEnabled(self.current_anim_idx>0)
        self.btn_next.setEnabled(self.current_anim_idx<t-1)
        if self.animations and self.current_anim_idx<t:
            a=self.animations[self.current_anim_idx]
            self.cur_label.setText(f"<b>#{a['index']+1} [{a['type']}]</b><br><span style='color:#888;font-size:10px;'>{a['content'][:80]}</span>")
        else:
            self.cur_label.setText("<b>Current:</b> None")
        self._refresh_obj()

    def _refresh_obj(self):
        for w in self.obj_widgets.values(): w.deleteLater()
        self.obj_widgets.clear()
        while self.obj_layout.count():
            it=self.obj_layout.takeAt(0)
            if it.widget(): it.widget().deleteLater()
        if self.cb_all.isChecked():
            names=list(self.all_objects.keys())
        elif self.animations and self.current_anim_idx<len(self.animations):
            names=self.animations[self.current_anim_idx]['objects']
        else:
            names=[]
        count=0
        for name in names:
            if name not in self.all_objects: continue
            w=ObjectControlWidget(self.all_objects[name])
            w.position_changed.connect(self._on_pos)
            w.requery_requested.connect(self._on_requery)
            w.font_size_changed.connect(self._on_font)
            w.color_changed.connect(self._on_color)
            w.scale_changed.connect(self._on_scale)
            w.opacity_changed.connect(self._on_opacity)
            w.width_changed.connect(self._on_width_live)
            w.height_changed.connect(self._on_height_live)
            self.obj_layout.addWidget(w); self.obj_widgets[name]=w; count+=1
        if not count:
            l=QLabel("No objects"); l.setStyleSheet("color:#888;padding:15px;"); self.obj_layout.addWidget(l)
        self.obj_counter.setText(str(count)); self.obj_layout.addStretch()

    def _prev(self):
        if self.current_anim_idx>0: self.current_anim_idx-=1; self._refresh_anim()

    def _next_ui(self):
        if self.current_anim_idx<len(self.animations)-1: self.current_anim_idx+=1; self._refresh_anim()

    # ═══════════════ OBJECT SIGNALS ═══════════════

    def _on_pos(self, name, pos):
        if name not in self.all_objects: return
        obj=self.all_objects[name]
        old=list(obj.get('position',[0,0,0]))
        obj['position']=list(pos); obj['position_modified']=True
        dx=pos[0]-old[0]; dy=pos[1]-old[1]; dz=pos[2]-old[2]
        live = self.manim_runner and self.manim_runner.is_alive()
        if live:
            self.manim_runner.send_command(f"{name}.move_to(np.array([{pos[0]},{pos[1]},{pos[2]}]))")
            self._log(f"→ {name}.move_to([{pos[0]:.2f},{pos[1]:.2f},{pos[2]:.2f}])")
        if obj.get('is_group') and (dx!=0 or dy!=0 or dz!=0):
            for ch in obj.get('all_descendants',[]):
                co=self.all_objects.get(ch)
                if not co: continue
                co['position']=[co['position'][0]+dx, co['position'][1]+dy, co['position'][2]+dz]
                co['position_modified']=True
                if ch in self.obj_widgets:
                    self.obj_widgets[ch].apply_delta(dx,dy,dz)

    def _on_requery(self, name):
        if self.manim_runner and self.manim_runner.is_alive():
            self.manim_runner.query_position(name)

    def _on_font(self, name, fs, ratio):
        if name in self.all_objects:
            self.all_objects[name]['font_size']=fs; self.all_objects[name]['font_modified']=True
        if self.manim_runner and self.manim_runner.is_alive():
            self.manim_runner.send_command(f"{name}.scale({ratio:.4f})")

    def _on_color(self, name, color):
        if name in self.all_objects:
            self.all_objects[name]['color']=color; self.all_objects[name]['color_modified']=True
        if self.manim_runner and self.manim_runner.is_alive():
            self.manim_runner.send_command(f'{name}.set_color("{color}")')

    def _on_scale(self, name, ratio):
        if name in self.all_objects:
            w = self.obj_widgets.get(name)
            if w:
                self.all_objects[name]['scale'] = w.object_scale
                self.all_objects[name]['scale_modified'] = w.scale_modified
        if self.manim_runner and self.manim_runner.is_alive():
            self.manim_runner.send_command(f"{name}.scale({ratio:.4f})")

    def _on_width_live(self, name, width):
        if name in self.all_objects:
            w = self.obj_widgets.get(name)
            if w:
                self.all_objects[name]['width'] = w.obj_width
                self.all_objects[name]['width_modified'] = True
                if w.obj_height:
                    self.all_objects[name]['height'] = w.obj_height
        if self.manim_runner and self.manim_runner.is_alive():
            self.manim_runner.send_command(f"{name}.stretch_to_fit_width({width:.4f})")

    def _on_height_live(self, name, height):
        if name in self.all_objects:
            w = self.obj_widgets.get(name)
            if w:
                self.all_objects[name]['height'] = w.obj_height
                self.all_objects[name]['height_modified'] = True
        if self.manim_runner and self.manim_runner.is_alive():
            self.manim_runner.send_command(f"{name}.stretch_to_fit_height({height:.4f})")

    def _on_opacity(self, name, val):
        if self.manim_runner and self.manim_runner.is_alive():
            self.manim_runner.send_command(f"{name}.set_opacity({val:.2f})")

    def _on_pos_received(self, name, pos):
        if name in self.all_objects:
            if not self.all_objects[name].get('position_modified'):
                self.all_objects[name]['position']=list(pos)
            self.all_objects[name]['position_queried']=True
        if name in self.obj_widgets:
            self.obj_widgets[name].update_live_position(pos)
        self._log(f"📍 {name}: ({pos[0]:.3f},{pos[1]:.3f},{pos[2]:.3f})")

    def _query_all(self):
        if not self.manim_runner or not self.manim_runner.is_alive():
            return
        queried=set()
        for idx in range(min(self.interactive_count+1, len(self.animations))):
            for name in self.animations[idx]['objects']:
                if name not in queried:
                    self.manim_runner.query_position(name); queried.add(name)

    def _on_safeplay_fallback(self, msg):
        self._log(f"🛡️ Layer 3: {msg}")

    # ═══════════════ APPLY TO CODE ═══════════════

    def _apply_to_code(self):
        code=self.code_editor.toPlainText()
        changes=0
        for name,obj in self.all_objects.items():
            w=self.obj_widgets.get(name)
            pm = (w.position_modified if w else obj.get('position_modified', False))
            if pm:
                pos = w.position if w else obj.get('position')
                if pos:
                    code=ManimCodeProcessor.update_object_position(code,name,pos,obj)
                    changes+=1
                    self._log(f"  ✏️ Pos: {name} → ({pos[0]:.2f},{pos[1]:.2f},{pos[2]:.2f})")
            fm = (w.font_modified if w else obj.get('font_modified', False))
            if fm:
                fs = w.font_size if w else obj.get('font_size')
                if fs: code=ManimCodeProcessor.update_object_font_size(code,name,fs); changes+=1
            cm = (w.color_modified if w else obj.get('color_modified', False))
            if cm:
                c = w.color if w else obj.get('color')
                if c and c.startswith('#'): code=ManimCodeProcessor.update_object_color(code,name,c); changes+=1
            sm = (w.scale_modified if w else obj.get('scale_modified', False))
            if sm:
                sv = w.object_scale if w else obj.get('scale', 1.0)
                code=ManimCodeProcessor.update_object_scale(code,name,sv); changes+=1
                self._log(f"  ✏️ Scale: {name} → {sv:.4f}")
            wm = (w.width_modified if w else obj.get('width_modified', False))
            if wm:
                wv = w.obj_width if w else obj.get('width')
                if wv is not None:
                    code=ManimCodeProcessor.update_object_width(code,name,wv,obj); changes+=1
            hm = (w.height_modified if w else obj.get('height_modified', False))
            if hm and not (obj.get('type') == 'Square' and wm):
                hv = w.obj_height if w else obj.get('height')
                if hv is not None:
                    code=ManimCodeProcessor.update_object_height(code,name,hv,obj); changes+=1
        if changes:
            self.code_editor.setPlainText(code)
            self._log(f"✓ Saved {changes} changes to code")
            for name in self.all_objects:
                for flag in ['position_modified','font_modified','color_modified',
                             'scale_modified','width_modified','height_modified']:
                    self.all_objects[name][flag] = False
                w=self.obj_widgets.get(name)
                if w:
                    w.position_modified=False; w.font_modified=False; w.color_modified=False
                    w.scale_modified=False; w.width_modified=False; w.height_modified=False
        else:
            self._log("ℹ No changes to save")
        return changes

    def _save_and_rerun(self):
        self._log("═══════════════════════════════════")
        self._log("💾 SAVE & RE-RUN")
        self._log("═══════════════════════════════════")
        n = self._apply_to_code()
        if n == 0: self._log("No changes, re-running anyway…")
        if self.manim_runner and self.manim_runner.is_alive():
            self.manim_runner.stop()
            self.manim_runner.wait(3000)
            self.manim_runner = None
        self._parse()
        self._run()

    def _run(self):
        code=self.code_editor.toPlainText()
        cn=self.class_input.text()
        if not cn:
            QMessageBox.warning(self,"Error","Specify Scene class name"); return
        self._log("─── Preparing with 3-Layer Safety ───")
        clean_code = ManimCodeProcessor.remove_interactive_embed(code)
        clean_code = ManimCodeProcessor.remove_safety_wrappers(clean_code)
        processed_code, warnings = ManimCodeProcessor.prepare_for_opengl_run(clean_code)
        self.layer_warnings = warnings
        for w in warnings: self._log(f"  🛡️ {w}")
        self.layer_info.setText(f"🛡️ {len(warnings)} fixes" if warnings else "🛡️ Layer 3 active")
        self.interactive_count=0; self.current_anim_idx=0; self._refresh_anim()
        self.manim_runner=ManimRunner(processed_code, cn)
        self.manim_runner.output_received.connect(self._log)
        self.manim_runner.error_received.connect(self._on_err)
        self.manim_runner.ready_for_input.connect(self._on_interactive)
        self.manim_runner.position_received.connect(self._on_pos_received)
        self.manim_runner.process_died.connect(self._on_died)
        self.manim_runner.safeplay_fallback.connect(self._on_safeplay_fallback)
        self.manim_runner.finished.connect(self._on_finished)
        self.manim_runner.start()
        self.btn_run.setEnabled(False); self.btn_stop.setEnabled(True)
        self.status_bar.showMessage("Running…"); self._log("▶ Starting Manim…")

    def _on_err(self, msg):
        self._log(f"ERR: {msg}")

    def _stop(self):
        if self.manim_runner: self.manim_runner.stop()

    def _on_interactive(self):
        self.btn_next_manim.setEnabled(True); self.btn_send.setEnabled(True); self.btn_requery.setEnabled(True)
        self.status_bar.showMessage(f"Interactive — after anim #{self.interactive_count+1}")
        self._log(f"─── Interactive (after anim #{self.interactive_count+1}) ───")
        if self.interactive_count < len(self.animations):
            self.current_anim_idx=self.interactive_count; self._refresh_anim()
        QTimer.singleShot(500, self._query_all)

    def _on_finished(self):
        self.btn_run.setEnabled(True); self.btn_stop.setEnabled(False)
        self.btn_next_manim.setEnabled(False); self.btn_send.setEnabled(False); self.btn_requery.setEnabled(False)
        self.status_bar.showMessage("Finished"); self._log("■ Finished")

    def _on_died(self):
        self._log("⚠️ Manim process exited unexpectedly")
        self.btn_run.setEnabled(True); self.btn_stop.setEnabled(False)
        self.btn_next_manim.setEnabled(False); self.btn_send.setEnabled(False)

    def _next_manim(self):
        self.interactive_count += 1
        if self.manim_runner and self.manim_runner.is_alive():
            self.manim_runner.send_command("exit")
            self._log("→ exit (advancing)")

    def _send_cmd(self):
        cmd=self.cmd_input.text()
        if cmd and self.manim_runner and self.manim_runner.is_alive():
            self.manim_runner.send_command(cmd); self._log(f"→ {cmd}"); self.cmd_input.clear()

    def _log(self, msg): self.console.append(msg)

    def _load(self):
        fn,_=QFileDialog.getOpenFileName(self,"Open","","Python (*.py);;All (*)")
        if fn:
            with open(fn,'r') as f: self.code_editor.setPlainText(f.read())
            self.status_bar.showMessage(f"Loaded: {fn}")

    def _save(self):
        fn,_=QFileDialog.getSaveFileName(self,"Save","","Python (*.py);;All (*)")
        if fn:
            with open(fn,'w') as f: f.write(self.code_editor.toPlainText())

    def _export(self):
        code = self.code_editor.toPlainText()
        code = ManimCodeProcessor.remove_interactive_embed(code)
        code = ManimCodeProcessor.remove_safety_wrappers(code)
        fn,_=QFileDialog.getSaveFileName(self,"Export","","Python (*.py);;All (*)")
        if fn:
            with open(fn,'w') as f: f.write(code)
            self._log(f"✓ Exported: {fn}")


def main():
    app=QApplication(sys.argv); app.setStyle('Fusion')
    p=QPalette()
    p.setColor(QPalette.ColorRole.Window,QColor(53,53,53))
    p.setColor(QPalette.ColorRole.WindowText,QColor(255,255,255))
    p.setColor(QPalette.ColorRole.Base,QColor(25,25,25))
    p.setColor(QPalette.ColorRole.AlternateBase,QColor(53,53,53))
    p.setColor(QPalette.ColorRole.ToolTipBase,QColor(255,255,255))
    p.setColor(QPalette.ColorRole.ToolTipText,QColor(255,255,255))
    p.setColor(QPalette.ColorRole.Text,QColor(255,255,255))
    p.setColor(QPalette.ColorRole.Button,QColor(53,53,53))
    p.setColor(QPalette.ColorRole.ButtonText,QColor(255,255,255))
    p.setColor(QPalette.ColorRole.BrightText,QColor(255,0,0))
    p.setColor(QPalette.ColorRole.Link,QColor(42,130,218))
    p.setColor(QPalette.ColorRole.Highlight,QColor(42,130,218))
    p.setColor(QPalette.ColorRole.HighlightedText,QColor(0,0,0))
    app.setPalette(p)
    win=ManimLiveEditor(); win.show()
    sys.exit(app.exec())

if __name__=="__main__":
    main(),, i give you a code first anyalysis it see what it does,, then i will tell you what proble it is creating,,
