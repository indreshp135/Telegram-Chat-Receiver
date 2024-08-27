<system.webServer>
  <rewrite>
    <outboundRules>
      <rule name="Add CSP header">
        <match serverVariable="RESPONSE_Content-Security-Policy" pattern=".*" />
        <action type="Rewrite" value="default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';" />
      </rule>
    </outboundRules>
  </rewrite>
</system.webServer>