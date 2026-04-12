# Chicago Multifamily Pipeline -- Data Sources Research

**Date:** 2026-04-11
**Purpose:** Comprehensive research on all free public data APIs and endpoints for the Chicago multifamily deal-sourcing pipeline. This document feeds the future `pipeline-data-sources-design.md` spec.
**Target geography:** Lincoln Park (community area 7) and Lakeview (community area 6)

---

## Source 1: Cook County Assessor Open Data (Socrata)

**Portal:** https://datacatalog.cookcountyil.gov
**API base:** `https://datacatalog.cookcountyil.gov/resource/{4x4-id}.json`
**Protocol:** Socrata SODA REST API (JSON, CSV, GeoJSON)
**Rate limits:** 1,000 requests/hour with app token; much lower without. Register free at https://dev.socrata.com
**Pagination:** Default 1,000 rows per request; use `$limit` and `$offset` parameters. Max `$limit` = 50,000.
**Filtering:** SoQL (Socrata Query Language) -- supports `$where`, `$select`, `$order`, `$group`, geographic functions like `within_polygon()`.

### 1A. Parcel Universe (Historical)
- **Dataset ID:** `nj4t-kc8j`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/nj4t-kc8j.json`
- **Coverage:** All Cook County parcels, 1999--present
- **Update frequency:** Monthly
- **Pipeline use:** Pre-filter (Stage 0) -- primary parcel identification and geographic filtering
- **Key fields:**
  - `pin`, `pin10` -- parcel identification
  - `year` -- assessment year
  - `class` -- property classification code (203 = 2-6 unit residential, 211 = 7+ unit, 318 = commercial, etc.)
  - `triad_name`, `triad_code` -- assessment triad
  - `township_name`, `township_code`
  - `nbhd_code` -- assessor neighborhood
  - `tax_code`
  - `zip_code`
  - `lon`, `lat` -- coordinates
  - `x_3435`, `y_3435` -- State Plane coordinates
  - `ward_num`, `ward_chicago_data_year`
  - `cook_municipality_name` -- filter for "CITY OF CHICAGO"
  - `census_tract_geoid`, `census_block_group_geoid`
  - `env_airport_noise_dnl`
  - `tax_tif_district_num`, `tax_tif_district_name` -- TIF overlay
  - `tax_park_district_name`
  - 79 fields total including census, school, and political district geographies

### 1B. Parcel Universe (Current Year Only)
- **Dataset ID:** `pabr-t5kh`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/pabr-t5kh.json`
- **Coverage:** Current-year parcels only with spatial and census geography
- **Pipeline use:** Faster alternative to historical dataset when only current data needed

### 1C. Parcel Addresses
- **Dataset ID:** `3723-97qp`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/3723-97qp.json`
- **Update frequency:** Updated December 2025
- **Pipeline use:** Pre-filter -- owner name, mailing address (absentee owner detection), property address
- **Key fields:**
  - `pin`, `pin10`, `year`
  - `prop_address_full`, `prop_address_city_name`, `prop_address_state`, `prop_address_zipcode_1`
  - `mail_address_name` -- owner name
  - `mail_address_full`, `mail_address_city_name`, `mail_address_state`, `mail_address_zipcode_1`
  - `owner_address_name`, `owner_address_full`, `owner_address_city_name`, `owner_address_state`, `owner_address_zipcode_1`
  - 18 fields total
- **Critical for:** Absentee owner detection (compare `prop_address` vs `mail_address`), LLC ownership detection (check `mail_address_name` for LLC/Corp patterns), owner name grouping for adjacent consolidation

### 1D. Single and Multi-Family Improvement Characteristics
- **Dataset ID:** `x54s-btds`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/x54s-btds.json`
- **Pipeline use:** Gate 1 -- building characteristics for development potential assessment
- **Key fields:**
  - `pin`, `year`, `card` (multi-improvement properties have multiple cards)
  - `class` -- property class
  - `township_code`
  - `tieback_proration_rate`
  - `cdu` -- condition/desirability/utility rating (AV = Average, etc.)
  - `pin_is_multicard`, `pin_num_cards` -- multi-building flag
  - `pin_is_multiland`, `pin_num_landlines`
  - `char_yrblt` -- year built
  - `char_bldg_sf` -- building square footage
  - `char_land_sf` -- lot size in square feet
  - `char_beds` -- number of bedrooms
  - `char_rooms` -- total rooms
  - `char_fbath` -- full bathrooms
  - `char_hbath` -- half bathrooms
  - `char_frpl` -- fireplaces
  - `char_type_resd` -- building type (2 Story, Split Level, etc.)
  - `char_cnst_qlty` -- construction quality (Average, Good, etc.)
  - `char_attic_fnsh`, `char_attic_type`
  - `char_gar1_att`, `char_gar1_area`, `char_gar1_size`, `char_gar1_cnst`
  - `char_bsmt`, `char_bsmt_fin`
  - `char_ext_wall` -- exterior wall material
  - `char_heat` -- heating type
  - `char_repair_cnd` -- repair condition
  - `char_roof_cnst`
  - `char_use` -- current use (Single-Family, Multi-Family, etc.)
  - `char_site`
  - `char_ncu` -- number of commercial units
  - `char_porch`
  - `char_air` -- AC type
  - `char_tp_plan`
  - 40 fields total
- **Critical for:** FAR calculation (char_bldg_sf / char_land_sf), age of building, condition assessment, current unit count, identifying underbuilt lots

### 1E. Assessed Values
- **Dataset ID:** `uzyt-m557`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/uzyt-m557.json`
- **Coverage:** 1999--present
- **Pipeline use:** Gate 1 -- land vs. building value ratio; Gate 2 -- detect declining values
- **Key fields:**
  - `pin`, `year`, `class`, `township_code`, `township_name`, `nbhd`
  - `mailed_bldg`, `mailed_land`, `mailed_tot` -- initial assessed values
  - `mailed_hie` -- homeowner improvement exemption
  - `certified_bldg`, `certified_land`, `certified_tot`, `certified_hie` -- post-appeal certified values
  - `board_bldg`, `board_land`, `board_tot`, `board_hie` -- Board of Review final values
  - 19 fields total
- **Critical for:** Land-to-total value ratio (high ratio = building is worth little relative to land = redevelopment candidate); year-over-year value trends

### 1F. Parcel Sales
- **Dataset ID:** `wvhk-k5uv`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/wvhk-k5uv.json`
- **Coverage:** 1999--present, sourced from IL Department of Revenue
- **Pipeline use:** Gate 1 -- comps/market value; Gate 2 -- hold duration, sale price trends
- **Key fields:**
  - `pin`, `year`, `township_code`, `nbhd`, `class`
  - `sale_date` -- date of sale
  - `sale_price` -- transaction price
  - `doc_no` -- document number (links to Recorder of Deeds)
  - `deed_type`, `mydec_deed_type` -- type of deed (Warranty, QuitClaim, Deed in Trust, etc.)
  - `seller_name`, `buyer_name` -- parties to the transaction
  - `is_multisale` -- flag if part of a multi-property transaction
  - `num_parcels_sale` -- number of parcels in the sale
  - `sale_filter_same_sale_within_365` -- flag for non-arm's-length transactions
  - `sale_filter_less_than_10k` -- flag for nominal sales
  - `sale_filter_deed_type` -- flag for non-standard deed types
  - 19 fields total
- **Critical for:** Hold duration (years since last sale), price per SF comps, identifying LLC-to-LLC transfers, filtering out non-arm's-length sales

### 1G. Parcel Proximity
- **Dataset ID:** `ydue-e5u3`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/ydue-e5u3.json`
- **Pipeline use:** Gate 1 -- TOD proximity, adjacent parcel detection
- **Key fields:**
  - `pin10`, `year`
  - `num_pin_in_half_mile` -- density proxy
  - `num_foreclosure_in_half_mile_past_5_years`
  - `num_foreclosure_per_1000_pin_past_5_years`
  - `airport_dnl_total` -- noise level
  - `nearest_new_construction_pin10`, `nearest_new_construction_dist_ft`
  - `nearest_stadium_name`, `nearest_stadium_dist_ft`
  - `nearest_vacant_land_pin10`, `nearest_vacant_land_dist_ft`
  - `nearest_neighbor_1_pin10`, `nearest_neighbor_1_dist_ft` (through neighbor 3)
- **Note:** This dataset does NOT include CTA/Metra distances; those are in the Parcel Universe dataset (fields `nearest_cta_stop_dist_ft`, `nearest_metra_stop_dist_ft`, etc.)

### 1H. Commercial Valuation Data
- **Dataset ID:** `csik-bsws`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/csik-bsws.json`
- **Pipeline use:** Gate 1 -- income-producing property analysis for larger multifamily
- **Key fields:**
  - `keypin`, `pins`, `year`, `township`, `class_es`, `address`
  - `adj_rent_sf` -- adjusted rent per SF
  - `bldgsf` -- building square footage
  - `caprate` -- capitalization rate
  - `excesslandval` -- excess land value
  - `exp` -- expense ratio
  - `finalmarketvalue`, `finalmarketvalue_sf` -- assessed market value
  - `incomemarketvalue` -- income approach value
  - `investmentrating` -- A through D rating
  - `landsf` -- land square footage
  - `noi` -- net operating income
  - `pgi` -- potential gross income
  - `property_type_use` -- property type (Retail-Storefront, Apartment, etc.)
  - `salecompmarketvalue_sf` -- sales comp value per SF
  - `vacancy` -- vacancy rate
  - `yearbuilt`
  - 23 fields total
- **Critical for:** Cap rate and NOI for income properties; vacancy rates; identifying underperforming assets

### 1I. Appeals
- **Dataset ID:** `y282-6ig3`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/y282-6ig3.json`
- **Pipeline use:** Gate 2 -- frequent appeals may signal dissatisfied owner
- **Coverage:** 1999--present

### 1J. Property Tax-Exempt Parcels
- **Dataset ID:** `vgzx-68gb`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/vgzx-68gb.json`
- **Pipeline use:** Pre-filter exclusion -- remove tax-exempt parcels

### 1K. Permits (CCAO)
- **Dataset ID:** `6yjf-dfxs`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/6yjf-dfxs.json`
- **Pipeline use:** Gate 1 -- recent permit activity signals active investment (or lack thereof)
- **Key fields:**
  - `pin`, `year`
  - `permit_number`, `local_permit_number`
  - `date_issued`
  - `status` -- OPEN, CLOSED
  - `amount` -- permit dollar amount
  - `municipality`, `township`
  - `mailing_address`
  - `applicant_name`
  - `work_description` -- text describing the work
  - 13 fields total
- **Note:** This is the CCAO's version of permits organized by PIN. Chicago DOB permits (ydr8-5enu) are more detailed.

### 1L. Residential Condominium Unit Characteristics
- **Dataset ID:** `3r7i-mrz4`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/3r7i-mrz4.json`
- **Pipeline use:** Condo-specific building characteristics (class 299, 399)

### 1M. Neighborhood Boundaries
- **Dataset ID:** `pcdw-pxtg`
- **Endpoint:** `https://datacatalog.cookcountyil.gov/resource/pcdw-pxtg.json`
- **Pipeline use:** Geographic reference for assessor neighborhood codes

---

## Source 2: Chicago Data Portal (Socrata)

**Portal:** https://data.cityofchicago.org
**API base:** `https://data.cityofchicago.org/resource/{4x4-id}.json`
**Protocol:** Socrata SODA REST API (same as Cook County)
**Rate limits:** Same Socrata rules -- 1,000 requests/hour with app token

### 2A. Building Permits
- **Dataset ID:** `ydr8-5enu`
- **Endpoint:** `https://data.cityofchicago.org/resource/ydr8-5enu.json`
- **Coverage:** 2006--present, updated daily
- **Pipeline use:** Gate 1 -- permit activity, development interest; Gate 2 -- expired permits = stalled projects
- **Key fields:**
  - `id`, `permit_` -- permit number
  - `permit_type` -- PERMIT - NEW CONSTRUCTION, PERMIT - RENOVATION/ALTERATION, PERMIT - WRECKING/DEMOLITION, PERMIT - SIGNS, etc.
  - `review_type`
  - `application_start_date`, `issue_date`
  - `processing_time`
  - `street_number`, `street_direction`, `street_name` -- address components
  - `work_description` -- free text describing the work (searchable for keywords like "demolition", "new construction", "add units")
  - `building_fee_paid`, `zoning_fee_paid`, `other_fee_paid`, `total_fee`
  - `reported_cost` -- total project cost reported by applicant
  - `contact_1_type`, `contact_1_name` -- contractor info
  - `community_area` -- can filter for 6 (Lakeview) and 7 (Lincoln Park)
  - `census_tract`, `ward`
  - `latitude`, `longitude`, `location`
  - 51 fields total
- **Critical for:** Filtering by `permit_type` = NEW CONSTRUCTION or WRECKING/DEMOLITION in target area to see active development; identifying expired permits on parcels; `reported_cost` as a proxy for project scale
- **Query example:** `$where=community_area in ('6','7') AND permit_type='PERMIT - NEW CONSTRUCTION'`

### 2B. Zoning Districts (Current Boundaries)
- **Dataset ID:** `7cve-jgbp`
- **Endpoint:** `https://data.cityofchicago.org/resource/7cve-jgbp.json`
- **Pipeline use:** Gate 1 -- determine what zoning applies to each parcel
- **Key fields:**
  - `the_geom` -- MultiPolygon geometry (GeoJSON)
  - `zone_type` -- numeric type code
  - `zone_class` -- zoning classification (RM-5, RT-4, B3-2, C1-3, etc.)
  - `edit_statu` -- ACTIVE status
  - `pd_prefix`, `pd_num` -- Planned Development identifiers
  - `shape_leng`, `shape_area` -- boundary measurements
  - 8 fields total
- **Critical for:** Spatial join to parcels to determine current zoning class; identifying PD overlays; RM/RT zones for multifamily by-right; B/C zones for mixed-use entitlement plays
- **Note:** This is a polygon dataset. Use spatial queries or download and join in geopandas.

### 2C. Zoning (Tabular)
- **Dataset ID:** `nifi-zqag`
- **Endpoint:** `https://data.cityofchicago.org/resource/nifi-zqag.json`
- **Pipeline use:** Same zoning data in tabular format
- **Key fields:** `the_geom`, `zone_type`, `zone_class`, `edit_statu`, `pd_prefix`, `pd_num`, `shape_leng`, `shape_area`

### 2D. Building Violations
- **Dataset ID:** `22u3-xenr`
- **Endpoint:** `https://data.cityofchicago.org/resource/22u3-xenr.json`
- **Coverage:** 2006--present
- **Pipeline use:** Gate 2 -- code violations signal deferred maintenance / distressed owner
- **Key fields:**
  - `id`, `violation_last_modified_date`, `violation_date`
  - `violation_code`, `violation_status` (OPEN, CLOSED, etc.)
  - `violation_description`
  - `violation_inspector_comments`
  - `violation_ordinance` -- specific ordinance violated
  - `inspector_id`, `inspection_number`
  - `inspection_status` -- PASSED, FAILED
  - `inspection_waived`
  - `inspection_category` -- PERMIT, COMPLAINT, PERIODIC, etc.
  - `department_bureau` -- CONSERVATION, ELEVATOR, NEW CONSTRUCTION, etc.
  - `address`, `street_number`, `street_direction`, `street_name`, `street_type`
  - `property_group` -- groups violations by property
  - `latitude`, `longitude`, `location`
  - 30 fields total
- **Critical for:** Count of open violations per property; violation age (days since violation_date); CONSERVATION violations = building condition issues; pattern of repeat violations = distressed owner

### 2E. Vacant and Abandoned Buildings (311 Reports)
- **Dataset ID:** `7nii-7srd` (current) / `gjy8-tm9a` (alternate)
- **Endpoint:** `https://data.cityofchicago.org/resource/7nii-7srd.json`
- **Coverage:** January 2010--present, updated daily
- **Pipeline use:** Gate 2 -- vacancy signals
- **Key fields per documentation:**
  - Address, whether building is vacant or occupied
  - Whether building is open or boarded
  - Entry point if building is open
  - Whether non-residents are occupying/using the building
  - Whether building appears dangerous or hazardous
  - Whether building is vacant due to fire
- **Note:** The JSON endpoint returned empty when tested; may need alternate dataset ID or different query approach. Try `7nii-7srd` first.

### 2F. City-Owned Land Inventory
- **Dataset ID:** `aksk-kvfp`
- **Endpoint:** `https://data.cityofchicago.org/resource/aksk-kvfp.json`
- **Pipeline use:** Identify adjacent city-owned lots (consolidation opportunity); exclude city-owned parcels from private owner outreach
- **Key fields:**
  - `id`, `pin` (14-digit format with dashes: 20-28-300-012-0000)
  - `address`
  - `managing_organization`
  - `property_status` -- "Owned by City"
  - `sales_status` -- "Application Closed", "Available", etc.
  - `sale_offering_status`
  - `sq_ft`, `square_footage_city_estimate`
  - `land_value`
  - `ward`, `community_area_number`, `community_area_name`
  - `zoning_classification` -- e.g., B1-2, RS-3
  - `zip_code`, `last_update`
  - `application_use`, `application_deadline`, `offer_round`, `application_url`
  - `latitude`, `longitude`
  - 30 fields total
- **Critical for:** Adjacent city lots that could be acquired through the city's Large Lots or ANLAP programs; zoning already assigned

### 2G. Affordable Housing Locations
- **Dataset ID:** `s6ha-ppgi`
- **Endpoint:** `https://data.cityofchicago.org/resource/s6ha-ppgi.json`
- **Pipeline use:** Reference -- identify existing affordable housing in target area
- **Key fields:**
  - `community_area`, `community_area_number`
  - `property_type` -- Multifamily, Senior, etc.
  - `property_name`, `address`, `zip_code`
  - `phone_number`, `management_company`
  - `units` -- unit count
  - `latitude`, `longitude`
  - 14 fields total

### 2H. CTA 'L' Rail Stations
- **Dataset ID:** `3tzw-cg4m`
- **Endpoint:** `https://data.cityofchicago.org/resource/3tzw-cg4m.json`
- **Pipeline use:** Gate 1 -- TOD proximity scoring (parcels within 1/4 mile of L station are TOD candidates with higher density allowances)
- **Key fields:**
  - `station_id`, `longname` -- station name
  - `lines` -- which L lines serve the station
  - `address`
  - `ada` -- ADA accessible
  - `pknrd` -- park and ride
  - `the_geom` -- Point geometry with coordinates
  - `point_x`, `point_y` -- State Plane coordinates
  - `legend` -- line color
  - 17 fields total
- **Critical for:** Distance calculation from each parcel to nearest L station. Chicago TOD ordinance allows density bonuses within 1/4 mile of rail stations. Lincoln Park and Lakeview are served by Brown, Red, and Purple lines.

### 2I. CTA 'L' Stops (Detailed)
- **Dataset ID:** `8pix-ypme`
- **Endpoint:** `https://data.cityofchicago.org/resource/8pix-ypme.json`
- **Pipeline use:** More granular stop-level data (each platform/entrance vs. station-level)

### 2J. Boundaries - Community Areas
- **Dataset ID:** `cauq-8yn6`
- **Endpoint:** `https://data.cityofchicago.org/resource/cauq-8yn6.json`
- **Pipeline use:** Geographic boundary polygons for filtering to Lincoln Park (7) and Lakeview (6)
- **Key fields:** `the_geom` (polygon), `area_numbe`, `community`, `shape_area`, `shape_len`

### 2K. TIF Districts Boundaries
- **Dataset ID:** `fz5x-7zak`
- **Endpoint:** `https://data.cityofchicago.org/resource/fz5x-7zak.json`
- **Pipeline use:** Gate 1 -- parcels in active TIF districts may benefit from TIF incentives for development

### 2L. TIF Funded Projects
- **Dataset ID:** `mex4-ppfc`
- **Endpoint:** `https://data.cityofchicago.org/resource/mex4-ppfc.json`
- **Pipeline use:** Reference -- see what TIF-funded projects are happening nearby (signals neighborhood investment)
- **Key fields:**
  - `tif_district`, `project_name`, `address`, `developer`
  - `project_description`
  - `cdc_date` -- Community Development Commission approval date
  - `approved_amount`, `total_project_cost`
  - `tif_subsidy_percentage`
  - `ward`, `community_area`
  - `latitude`, `longitude`
  - 24 fields total

### 2M. Socioeconomic Hardship Index
- **Dataset ID:** `kn9c-c2s2`
- **Endpoint:** `https://data.cityofchicago.org/resource/kn9c-c2s2.json`
- **Pipeline use:** Background context for community area economics
- **Key fields:**
  - `ca` -- community area number
  - `community_area_name`
  - `percent_of_housing_crowded`
  - `percent_households_below_poverty`
  - `percent_aged_16_unemployed`
  - `percent_aged_25_without_high_school_diploma`
  - `per_capita_income_`
  - `hardship_index`

---

## Source 3: Cook County Clerk -- Delinquent Property Taxes

**Portal:** https://taxdelinquent.cookcountyclerkil.gov and https://www.cookcountyclerkil.gov/property-taxes/delinquent-property-tax-search
**Protocol:** Web search form + downloadable bulk file (NOT a REST API)
**Access method:** Bulk CSV download (full 20-year delinquent property file, updated monthly)

### What's available:
- PIN-level delinquency data going back 20 years
- Field descriptions for the downloadable file are available on the Clerk's website
- Monthly updates
- Searchable by PIN or address through the web interface

### Access approach for pipeline:
- **Download the full delinquent property file** monthly (bulk CSV)
- Filter to target PINs from pre-filter stage
- No API -- this is a file download, not a queryable endpoint
- The web search form at taxdelinquent.cookcountyclerkil.gov can be scraped PIN-by-PIN as a fallback, but the bulk file is the correct approach

### Pipeline use:
- **Gate 2** -- tax delinquency is a primary motivation signal
- Cross-reference with parcel universe PINs to flag delinquent parcels
- Duration of delinquency (years) is a weighted signal

---

## Source 4: Cook County Treasurer

**Portal:** https://www.cookcountytreasurer.com
**Protocol:** Web application only -- NO public API
**Access method:** PIN-by-PIN lookup through web interface; would require scraping

### What's available per PIN:
- 5-year history of tax amounts billed
- Current payment status
- Whether delinquent taxes have been sold at tax sale
- Exemptions currently applied (homeowner, senior, etc.)
- Tax bill download (PDF)
- Refund availability

### Access approach for pipeline:
- **Not recommended as a primary source** -- no API, scraping is fragile
- Use the Cook County Clerk bulk delinquent file (Source 3) for delinquency data instead
- Use Cook County Assessor Assessed Values (Source 1E) for tax assessment data
- The Treasurer site is useful for manual verification of individual parcels during the review stage

---

## Source 5: Cook County Property Tax Portal

**Portal:** https://www.cookcountypropertyinfo.com
**Protocol:** Web application only -- NO public API
**Access method:** Search by PIN or address

### What's available:
- Consolidated view pulling from Assessor, Treasurer, and Recorder of Deeds
- 5-year tax bill history
- Assessment history
- Exemption details
- Transfer/sale history
- Tax payment status

### Access approach for pipeline:
- **Not a data source for the pipeline** -- this is a consumer-facing portal consolidating data that's available via the individual agency APIs/downloads
- Useful as a manual verification tool in the Review UI ("Open in Property Tax Portal" link per parcel)
- All underlying data is better accessed through Sources 1, 3, and 6

---

## Source 6: Cook County Recorder of Deeds (now Cook County Clerk)

**Portal:** https://www.cookcountyclerkil.gov/recordings/search-recordings
**Protocol:** Web search form only -- NO public API
**Access method:** Search by address, PIN, grantor, grantee, document number

### What's available:
- Property transfer documents going back to 1985 (online)
- Documents from ~1970--1985 available in-person
- Deed type, grantor/grantee names, recording date, document number

### Access approach for pipeline:
- **Primary sale history data comes from Cook County Assessor Parcel Sales** (Source 1F, dataset `wvhk-k5uv`) which is sourced from IL Department of Revenue and includes `doc_no` that links to Recorder documents
- The Recorder's search is useful for manual deep-dives on specific parcels (e.g., looking at the actual deed to identify trust beneficiaries)
- **Not recommended for automated pipeline use** -- no API, and the Parcel Sales dataset covers the same ground
- Could be scraped PIN-by-PIN as enrichment, but the effort/fragility ratio is poor

---

## Source 7: Illinois Secretary of State -- LLC/Business Entity Search

**Portal:** https://apps.ilsos.gov/businessentitysearch/ (replaces retired Cyberdrive Illinois)
**Protocol:** Web search form only -- NO public API
**Access method:** Search by entity name, partial name, keyword, or file number

### What's available:
- LLC status (Active, Inactive, Dissolved, etc.)
- File number and date of incorporation
- Registered agent name and address
- Principal office address
- Date of last annual report
- Filing history

### Access approach for pipeline:
- **Gate 2 / Enrichment** -- for LLCs identified as property owners, look up registered agent to find the actual person behind the LLC
- **No bulk download allowed** -- the database "may not be used to copy or download bulk information"
- Must be queried one entity at a time
- **Scraping approach:** For each property owner identified as an LLC from the Assessor data, query the SOS search to get registered agent info
- Consider using **OpenCorporates** as an alternative (see Source 8)
- Rate limiting: Unknown, but be conservative (1 request per 3--5 seconds recommended)
- This is an **enrichment-stage** source, not pre-filter or gate stage

### Important limitations:
- No programmatic API
- Terms prohibit bulk downloads
- Must search by exact or partial entity name
- Results may include multiple entities with similar names
- Registered agent =/= beneficial owner in all cases

---

## Source 8: OpenCorporates

**Portal:** https://opencorporates.com
**API:** https://api.opencorporates.com (v0.4.8)
**Protocol:** REST API (JSON)

### What's available:
- 2.8M+ Illinois company records (LLCs, Corps, LPs, etc.)
- Company name, status, incorporation date, jurisdiction
- Officers and registered agents
- Filing history

### Pricing:
- **Free tier:** Web search only, limited API access
- **Paid API:** Starting GBP 2,250/year for commercial use
- **Free bulk access:** Available for academics, NGOs, journalists, and nonprofits

### Access approach for pipeline:
- **Alternative to scraping IL SOS** for LLC lookups
- Probably too expensive for this project's scale
- Better to scrape IL SOS directly for the small number of enrichment lookups needed (20--50 per pipeline run)

---

## Source 9: Chicago Zoning Map Application

**Portal:** https://gisapps.cityofchicago.org/zoning/
**Protocol:** Web application with ArcGIS backend
**Access method:** Interactive map; not a standard REST API

### What's available:
- Current zoning classification for any address/parcel
- Planned Development boundaries and ordinance numbers
- Zoning map amendment history (proposed changes to City Council)
- Links to zoning ordinance text

### Access approach for pipeline:
- **The Data Portal zoning dataset (Source 2B/2C) provides the same zoning boundary data** via API
- The zoning map application is useful for manual verification and for checking zoning amendment history (which is NOT in the data portal datasets)
- For zoning change history, this may need to be scraped from the application or from City Council ordinance records

---

## Source 10: Second City Zoning

**Portal:** https://secondcityzoning.org
**Protocol:** Web application (open-source, built on Chicago open data)

### What's available:
- Interactive zoning map with plain-language explanations of what each zone allows
- FAR, density, height limits, and setback requirements for each zoning class
- Built on the same Chicago Data Portal zoning data

### Access approach for pipeline:
- **Not a data source** -- it's a visualization layer
- However, it's an excellent reference for building a **zoning lookup table** that maps zone_class to FAR, max height, max units, and setback requirements
- This lookup table is needed for Gate 1 to calculate as-of-right development potential

---

## Source 11: CTA GTFS Data

**Portal:** https://www.transitchicago.com/developers/gtfs/
**Protocol:** GTFS static feed (ZIP file with CSV files)

### What's available:
- Complete CTA system data: all bus stops, rail stations, routes, schedules
- stops.txt contains lat/lng coordinates for every stop
- Standard GTFS format used by Google Maps, Apple Maps, etc.

### Access approach for pipeline:
- **Download stops.txt** from GTFS feed for rail station coordinates
- Or use the CTA L Stations dataset from the Data Portal (Source 2H) which is simpler
- The Data Portal version is sufficient for TOD proximity calculations

---

## Source 12: Cook County GIS (CookViewer / ArcGIS Hub)

**Portal:** https://maps.cookcountyil.gov/cookviewer/ and https://hub-cookcountyil.opendata.arcgis.com
**Protocol:** ArcGIS REST services + web map viewer

### What's available:
- Parcel boundaries (polygon geometry)
- Aerial imagery
- Property information overlays
- Multiple GIS layers including parcels, tax districts, municipalities

### Access approach for pipeline:
- **Parcel boundaries** may be needed if building a parcel-level map view
- The ArcGIS Hub may expose REST endpoints for parcel polygons
- Not needed for initial pipeline -- the Assessor data provides point coordinates per parcel
- Could be useful later for building footprint analysis and lot coverage calculations

---

## Summary: Data Source to Pipeline Stage Mapping

### Pre-Filter (Stage 0) -- Primary sources:
| Source | Dataset | What it provides |
|---|---|---|
| Cook County Assessor | Parcel Universe (`nj4t-kc8j` or `pabr-t5kh`) | PIN, lat/lng, class code, ward, zip, TIF overlay |
| Cook County Assessor | Parcel Addresses (`3723-97qp`) | Owner name, mailing address, property address |
| Cook County Assessor | Improvement Characteristics (`x54s-btds`) | Lot size (char_land_sf), building SF, year built, use type |
| Cook County Assessor | Tax-Exempt Parcels (`vgzx-68gb`) | Exclusion list |
| Chicago Data Portal | Community Area Boundaries (`cauq-8yn6`) | Geographic filter polygon |

### Gate 1 -- Development Potential:
| Source | Dataset | What it provides |
|---|---|---|
| Cook County Assessor | Improvement Characteristics (`x54s-btds`) | Building SF, lot SF, year built, condition, unit count |
| Cook County Assessor | Assessed Values (`uzyt-m557`) | Land vs building value ratio |
| Cook County Assessor | Commercial Valuation (`csik-bsws`) | Cap rate, NOI, vacancy for income properties |
| Cook County Assessor | Parcel Sales (`wvhk-k5uv`) | Recent sale prices for comps |
| Cook County Assessor | Parcel Proximity (`ydue-e5u3`) | Nearby new construction, foreclosures |
| Chicago Data Portal | Zoning Districts (`7cve-jgbp`) | Current zoning class (needed for FAR/density calc) |
| Chicago Data Portal | Building Permits (`ydr8-5enu`) | Recent permit activity, new construction nearby |
| Chicago Data Portal | CTA L Stations (`3tzw-cg4m`) | TOD proximity |
| Chicago Data Portal | TIF Districts (`fz5x-7zak`) | TIF incentive availability |
| Chicago Data Portal | City-Owned Land (`aksk-kvfp`) | Adjacent city lots for consolidation |

### Gate 2 -- Motivation Scoring:
| Source | Dataset | What it provides |
|---|---|---|
| Cook County Assessor | Parcel Sales (`wvhk-k5uv`) | Hold duration, LLC-to-LLC transfers |
| Cook County Assessor | Parcel Addresses (`3723-97qp`) | Absentee owner flag, LLC ownership detection |
| Cook County Assessor | Appeals (`y282-6ig3`) | Frequent appeals = dissatisfied owner |
| Cook County Assessor | Assessed Values (`uzyt-m557`) | Declining values over time |
| Cook County Clerk | Delinquent Tax File (bulk download) | Tax delinquency flag + duration |
| Chicago Data Portal | Building Violations (`22u3-xenr`) | Code violations count and severity |
| Chicago Data Portal | Vacant Buildings (`7nii-7srd`) | Vacancy reports |

### Enrichment (Stage 3):
| Source | Dataset | What it provides |
|---|---|---|
| IL Secretary of State | Business Entity Search (scrape) | LLC registered agent = real person behind LLC |
| BatchSkipTracing | Paid API | Phone, email for owner contact |

---

## Rate Limit and Pagination Summary

| Source | Protocol | Rate Limit | Pagination | Auth Required |
|---|---|---|---|---|
| Cook County Assessor (Socrata) | REST/JSON | 1,000 req/hr with token | `$limit` + `$offset`, max 50k rows | App token (free) |
| Chicago Data Portal (Socrata) | REST/JSON | 1,000 req/hr with token | `$limit` + `$offset`, max 50k rows | App token (free) |
| Cook County Clerk (delinquent taxes) | Bulk CSV download | N/A (file download) | N/A | None |
| Cook County Treasurer | Web scrape | Unknown; be conservative | One PIN at a time | None |
| Cook County Recorder (Clerk) | Web scrape | Unknown; be conservative | One PIN at a time | None |
| IL Secretary of State | Web scrape | Unknown; ~1 req/3-5 sec | One entity at a time | None |
| Cook County GIS (ArcGIS) | REST/JSON | Unknown | Standard ArcGIS pagination | None |

---

## Key Query Examples (SoQL)

### Get all multifamily parcels in Lincoln Park and Lakeview:
```
GET https://datacatalog.cookcountyil.gov/resource/nj4t-kc8j.json?
  $where=cook_municipality_name='CITY OF CHICAGO'
    AND year=2025
    AND (ward_num=43 OR ward_num=44 OR ward_num=46)
  &$limit=50000
```
Note: Ward-based filtering is imprecise for community areas. Better approach: download community area polygons from Source 2J and do a spatial filter in Python with geopandas.

### Get all sales in target area since 2015:
```
GET https://datacatalog.cookcountyil.gov/resource/wvhk-k5uv.json?
  $where=sale_date > '2015-01-01'
    AND sale_price > 10000
    AND pin starts_with '14'
  &$limit=50000
```
Note: PINs starting with '14' cover much of the North Side; refine with township_code or join to parcel universe.

### Get new construction permits in Lakeview/Lincoln Park:
```
GET https://data.cityofchicago.org/resource/ydr8-5enu.json?
  $where=community_area in ('6','7')
    AND permit_type='PERMIT - NEW CONSTRUCTION'
    AND issue_date > '2020-01-01'
  &$limit=50000
```

### Get open building violations:
```
GET https://data.cityofchicago.org/resource/22u3-xenr.json?
  $where=violation_status='OPEN'
    AND latitude > 41.91 AND latitude < 41.96
    AND longitude > -87.66 AND longitude < -87.63
  &$limit=50000
```

---

## Sources Not Available as APIs (Manual/Scrape Only)

| Source | Access Method | Usefulness | Recommendation |
|---|---|---|---|
| Cook County Treasurer | Web form, PIN-by-PIN | Tax payment status, exemptions | Skip; use Clerk bulk file for delinquency |
| Cook County Property Tax Portal | Web form | Consolidated view | Skip; underlying data available via APIs |
| Cook County Recorder/Clerk | Web form | Deed documents, transfer history | Skip; use Assessor Parcel Sales instead |
| IL Secretary of State | Web form | LLC registered agent | Scrape at enrichment stage only (~20-50 lookups) |
| Chicago Zoning Map App | ArcGIS web app | Zoning amendment history | Scrape selectively; current zoning via Data Portal |
| Listing status (LoopNet, etc.) | Web scrape | Active listings = broker route | Defer to enrichment; terms may prohibit scraping |

---

## Data Not Freely Available (Gaps)

1. **Zoning change/amendment history:** No dataset of past rezoning actions. Would need to scrape City Council ordinance records or the zoning map application. Important for entitlement track (where has rezoning been approved recently?).

2. **Building footprints with height/stories:** The Chicago building footprints dataset (`hz9b-7nh8`) exists but returned empty in testing. May need to access via the GeoJSON/Shapefile download rather than JSON API. Would provide building envelope data for FAR analysis.

3. **Rental registration / vacancy rates:** No public dataset of rental registrations or unit-level vacancy. The Commercial Valuation dataset has vacancy rates but only for properties the assessor classifies as commercial.

4. **MLS/listing data:** No free API. LoopNet, CoStar, Crexi all require paid subscriptions or prohibit scraping. For listing/broker check, may need to rely on manual search or BatchSkipTracing enrichment data.

5. **Property condition photos:** No Street View API integration (per design constraint). Google Maps link is the fallback.

6. **Detailed zoning allowances table:** No API that returns "for zone RM-5, FAR = 2.0, max height = 47ft, max density = 1 unit per 1,000 sf lot area." This must be built as a static lookup table from the Chicago Zoning Ordinance or Second City Zoning.
