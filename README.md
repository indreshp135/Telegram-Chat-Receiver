Mandatory Analysis Framework:

        1. **Blacklist & Shell Company Check**
        - Flag shell company patterns: anonymous ownership, nominee directors, lack of physical address
        - Verify entity registration status and dissolution records

        2. **Sanctions Screening**
        - Cross-check all parties against global sanctions lists
        - Highlight full/partial name matches with SDN lists

        3. **PEP & Associates Analysis**
        - Identify PEP status (current/historical)
        - Map close associates through family/ownership relationships
        - Calculate ownership percentages in connected entities

        4. **Jurisdictional Risk** 
        - FATF greylist/blacklist status
        - High-risk geography patterns (tax havens, conflict zones)
        - WIKIDATA-documented sanctions history

        5. **Adverse Media**
        - Recent negative coverage (last 3 years)
        - Fraud/regulatory action mentions
        - Industry-specific risk patterns

        6. **Transaction Contextualization** 
        - Neo4J historical analysis: counterparty relationships, network clusters
        - Pattern deviations from historical behavior
        - High-risk transaction types (layering, structuring, round amounts)

        7. **Composite Risk Scoring**
        - Weighted evaluation of all factors
        - Explicit confidence scoring for missing data

        For unavailable data, state gaps but proceed with available information.

        Deliver assessment in this JSON structure:
        {{
        "extracted_entities": ["string"],
        "entity_types": ["string"],
        "risk_score": float (Calculate an overall risk score between 0 and 1 (0 = low risk, 1 = high risk)),
        "supporting_evidence": ["string"],
        "confidence_score": float,
        "reason": "Multi-factor analysis: [1-2 sentence summary]. Highest risk contributors: [top factors]"
        }}

        The "extracted_entities" should include all organizations and people from the data. 
        The "entity_types" should reflect the type of each entity.
        The "supporting_evidence" should list the key pieces of evidence for your risk assessment.
        The "confidence_score" should reflect how confident you are in your assessment.
        The "reason" should provide a deta