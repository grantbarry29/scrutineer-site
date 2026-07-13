{{- /* Raw-Markdown edition of every regular page, served at the pretty
     URL + ".md" (e.g. /docs/faq.md). {{< details >}} shortcodes become
     plain "##" headings; root-relative links become absolute so the
     file stands alone when fetched by an agent. The shortcode
     delimiters are built with printf so this template contains no
     literal action delimiters. */ -}}
{{- $ld := printf "%c%c" 123 123 -}}
{{- $rd := printf "%c%c" 125 125 -}}
{{- $md := .RawContent -}}
{{- $md = replaceRE (printf `%s<\s*details\b[^>]*?summary="([^"]+)"[^>]*>%s` $ld $rd) "## $1" $md -}}
{{- $md = replaceRE (printf `%s<\s*/\s*details\s*>%s` $ld $rd) "" $md -}}
{{- $md = replaceRE `\]\(/` (printf "](%s" site.BaseURL) $md -}}
# {{ .Title }}

{{ with .Summary }}> {{ . | plainify | htmlUnescape | chomp }}

{{ end -}}
{{ $md -}}
