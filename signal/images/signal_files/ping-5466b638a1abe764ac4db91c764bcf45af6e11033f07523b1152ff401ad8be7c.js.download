(function() {
	'use strict';

	if (window.signalcounter && window.signalcounter.vars)  // Compatibility
		window.signalcounter = window.signalcounter.vars
	else
		window.signalcounter = window.signalcounter || {}

	// Get all data we're going to send off to the counter endpoint.
	var get_data = function(vars) {
		var data = {
			p: (vars.path     === undefined ? signalcounter.path     : vars.path),
			r: (vars.referrer === undefined ? signalcounter.referrer : vars.referrer),
			t: (vars.title    === undefined ? signalcounter.title    : vars.title),
			e: !!(vars.event || signalcounter.event),
			s: [window.screen.width, window.screen.height, (window.devicePixelRatio || 1)],
			b: is_bot(),
			q: location.search,
		}

		var rcb, pcb, tcb  // Save callbacks to apply later.
		if (typeof(data.r) === 'function') rcb = data.r
		if (typeof(data.t) === 'function') tcb = data.t
		if (typeof(data.p) === 'function') pcb = data.p

		if (is_empty(data.r)) data.r = document.referrer
		if (is_empty(data.t)) data.t = document.title
		if (is_empty(data.p)) {
			var loc = location,
			    c = document.querySelector('link[rel="canonical"][href]')
			if (c) {  // May be relative or point to different domain.
				var a = document.createElement('a')
				a.href = c.href
				if (a.hostname.replace(/^www\./, '') === location.hostname.replace(/^www\./, ''))
					loc = a
			}
			data.p = (loc.pathname + loc.search) || '/'
		}

		if (rcb) data.r = rcb(data.r)
		if (tcb) data.t = tcb(data.t)
		if (pcb) data.p = pcb(data.p)
		return data
	}

	// Check if a value is "empty" for the purpose of get_data().
	var is_empty = function(v) { return v === null || v === undefined || typeof(v) === 'function' }

	// See if this looks like a bot; there is some additional filtering on the
	// backend, but these properties can't be fetched from there.
	var is_bot = function() {
		// Headless browsers are probably a bot.
		var w = window, d = document
		if (w.callPhantom || w._phantom || w.phantom)
			return 150
		if (w.__nightmare)
			return 151
		if (d.__selenium_unwrapped || d.__webdriver_evaluate || d.__driver_evaluate)
			return 152
		if (navigator.webdriver)
			return 153
		return 0
	}

	// Object to urlencoded string, starting with a ?.
	var urlencode = function(obj) {
		var p = []
		for (var k in obj)
			if (obj[k] !== '' && obj[k] !== null && obj[k] !== undefined && obj[k] !== false)
				p.push(encodeURIComponent(k) + '=' + encodeURIComponent(obj[k]))
		return '?' + p.join('&')
	}

	// Get the endpoint to send requests to.
	var get_endpoint = function() {
		var s = document.querySelector('script[data-signalcounter]');
		if (s && s.dataset.signalcounter)
			return s.dataset.signalcounter
		return (signalcounter.endpoint || window.counter)  // counter is for compat; don't use.
	}

	// Filter some requests that we (probably) don't want to count.
	signalcounter.filter = function() {
		if ('visibilityState' in document && (document.visibilityState === 'prerender' || document.visibilityState === 'hidden'))
			return 'visibilityState'
		if (!signalcounter.allow_frame && location !== parent.location)
			return 'frame'
//		if (!signalcounter.allow_local && location.hostname.match(/(localhost$|^127\.|^10\.|^172\.(1[6-9]|2[0-9]|3[0-1])\.|^192\.168\.)/))
//			return 'localhost'
		if (!signalcounter.allow_local && location.protocol === 'file:')
			return 'localfile'
		if (localStorage && localStorage.getItem('skipsc') === 't')
			return 'disabled with #toggle-signalcounter'
		return false
	}

	// Get URL to send to SignalCounter.
	window.signalcounter.url = function(vars) {
		var data = get_data(vars || {})
		if (data.p === null)  // null from user callback.
			return
		data.rnd = Math.random().toString(36).substr(2, 5)  // Browsers don't always listen to Cache-Control.

		var endpoint = get_endpoint()
		if (!endpoint) {
			if (console && 'warn' in console)
				console.warn('signalcounter: no endpoint found')
			return
		}

		return endpoint + urlencode(data)
	}

	// Count a hit.
	window.signalcounter.count = function(vars) {
		var f = signalcounter.filter()
		if (f) {
			if (console && 'log' in console)
				console.warn('signalcounter: not counting because of: ' + f)
			return
		}

		var url = signalcounter.url(vars)
		if (!url) {
			if (console && 'log' in console)
				console.warn('signalcounter: not counting because path callback returned null')
			return
		}

		var img = document.createElement('img')
		img.src = url
		img.style.position = 'absolute'  // Affect layout less.
		img.setAttribute('alt', '')
		img.setAttribute('aria-hidden', 'true')

		var rm = function() { if (img && img.parentNode) img.parentNode.removeChild(img) }
		setTimeout(rm, 3000)  // In case the onload isn't triggered.
		img.addEventListener('load', rm, false)
		document.body.appendChild(img)
	}

	// Get a query parameter.
	window.signalcounter.get_query = function(name) {
		var s = location.search.substr(1).split('&')
		for (var i = 0; i < s.length; i++)
			if (s[i].toLowerCase().indexOf(name.toLowerCase() + '=') === 0)
				return s[i].substr(name.length + 1)
	}

	// Track click events.
	window.signalcounter.bind_events = function() {
		if (!document.querySelectorAll)  // Just in case someone uses an ancient browser.
			return

		var send = function(elem) {
			return function() {
				signalcounter.count({
					event:    true,
					path:     (elem.dataset.signalcounterClick || elem.name || elem.id || ''),
					title:    (elem.dataset.signalcounterTitle || elem.title || (elem.innerHTML || '').substr(0, 200) || ''),
					referrer: (elem.dataset.signalcounterReferrer || elem.dataset.signalcounterReferral || ''),
				})
			}
		}

		Array.prototype.slice.call(document.querySelectorAll("*[data-signalcounter-click]")).forEach(function(elem) {
			if (elem.dataset.signalcounterBound)
				return
			var f = send(elem)
			elem.addEventListener('click', f, false)
			elem.addEventListener('auxclick', f, false)  // Middle click.
			elem.dataset.signalcounterBound = 'true'
		})
	}

	// Make it easy to skip your own views.
	if (location.hash === '#toggle-signalcounter')
		if (localStorage.getItem('skipsc') === 't') {
			localStorage.removeItem('skipsc', 't')
			alert('SignalCounter tracking is now ENABLED in this browser.')
		}
		else {
			localStorage.setItem('skipgc', 't')
			alert('SignalCounter tracking is now DISABLED in this browser until ' + location + ' is loaded again.')
		}

	if (!signalcounter.no_onload) {
		var DOMReady = function(callback) {
			document.readyState === "interactive" || document.readyState ==="complete" ? callback() : document.addEventListener("load", callback);
		};

		DOMReady(function() {
			signalcounter.count()
//			if (!signalcounter.no_events)
//				signalcounter.bind_events()
		});
/*
		var go = function() {
			signalcounter.count()
			if (!signalcounter.no_events)
				signalcounter.bind_events()
		}

		if (document.body === null)
			document.addEventListener('DOMContentLoaded', function() { go() }, false)
		else
			go()
*/
	}
})();
