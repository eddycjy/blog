{{ define "head" }}
	{{ if .Params.featuredImg -}}
	<style>.bg-img {background-image: url('{{.Params.featuredImg}}');}</style>
	{{- else if .Params.images -}}
		{{- range first 1 .Params.images -}}
		<style>.bg-img {background-image: url('{{. | absURL}}');}</style>
		{{- end -}}
	{{- end -}}
{{ end }}

{{ define "header" }}
{{ partial "header.html" . }}
{{ end }}

{{ define "main" }}
	{{- if (or .Params.images .Params.featuredImg) }}
	<div class="bg-img"></div>
	{{- end }}
	<main class="site-main section-inner thin animated fadeIn faster">
		<h1>{{ .Title }}</h1>
		<div class="content">
			{{ .Content | replaceRE "(<h[1-6] id=\"([^\"]+)\".+)(</h[1-6]+>)" `${1}<a href="#${2}" class="anchor" aria-hidden="true"><svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M15 7h3a5 5 0 0 1 5 5 5 5 0 0 1-5 5h-3m-6 0H6a5 5 0 0 1-5-5 5 5 0 0 1 5-5h3"></path><line x1="8" y1="12" x2="16" y2="12"></line></svg></a>${3}` | safeHTML }}
		</div>
		{{- if .Params.comments }}
		<div id="comments" class="thin">
			{{ partial "comments.html" . }}
		</div>
		{{- end }}
	</main>
{{ end }}

{{ define "footer" }}
{{ partialCached "footer.html" . }}
{{ end }}
