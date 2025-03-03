<script lang="ts">
	import { onMount, onDestroy } from "svelte";
	import ClassModal from "../components/ClassModal.svelte";
	import AddClassModal from "../components/AddClassModal.svelte";
	import TimeSelectionModal from "../components/TimeSelectionModal.svelte";
	import ProfessorModal from "../components/ProfessorModal.svelte";
	import ScheduleView from "../components/ScheduleView.svelte";
	import "../styles/page.css";
	import "./styles/page-styles.css";
	import "../styles/global.css"; // Import global styles

	// Import all logic from the separate file
	import * as logic from "./page-logic";
	import {
		availableClasses,
		addedClasses,
		prohibitedTimes,
		prohibitedProfessors,
		colorMap,
		showTimeSelectionModal,
		showAddClassModal,
		generatingSchedules,
		currentAmountLoaded,
		showClassModals,
		showProfessorModals,
		schedules,
		sortOption,
		error,
		viewProfessorInfo,
		generatedSchedules,
		addTimeRestriction,
		addMultipleTimeRestrictions,
	} from "./page-logic";

	let scrollContainer: HTMLElement | null = null;

	// Reactive statements to manage state
	$: sortedSchedules = logic.sortSchedules($schedules);

	// Assign colors to added classes when they change
	$: {
		$addedClasses.forEach((className, index) => {
			if (!$colorMap.has(className)) {
				$colorMap.set(className, logic.generateColor(index));
			}
		});
	}

	onMount(async () => {
		// Fetch available classes
		try {
			const url = new URL("https://api.umd.io/v1/courses/list");
			url.searchParams.set("sort", "course_id,-credits");
			url.searchParams.set("semester", "202501");

			const response = await fetch(url);
			const data = await response.json();
			$availableClasses = data.map(
				(item: { course_id: string }) => item.course_id,
			);
		} catch (err) {
			$error =
				err instanceof Error ? err.message : "Failed to fetch classes";
			console.error("Error fetching classes:", err);
		}

		// Initialize class-professor tracking
		$addedClasses.forEach((className) => {
			logic.fetchClassDetails(className);
		});
	});
</script>

<div class="app-container">
	<header class="app-header">
		<div class="header-content">
			<h1 class="app-title">UMD Scheduler</h1>
			<div class="header-actions">
				<button
					class="btn btn-outline"
					on:click={() => ($showTimeSelectionModal = true)}
				>
					<svg
						xmlns="http://www.w3.org/2000/svg"
						width="16"
						height="16"
						viewBox="0 0 24 24"
						fill="none"
						stroke="currentColor"
						stroke-width="2"
					>
						<circle cx="12" cy="12" r="10"></circle>
						<polyline points="12 6 12 12 16 14"></polyline>
					</svg>
					Add Time Restriction
				</button>
				<button
					class="btn btn-outline"
					on:click={() => ($showAddClassModal = true)}
				>
					<svg
						xmlns="http://www.w3.org/2000/svg"
						width="16"
						height="16"
						viewBox="0 0 24 24"
						fill="none"
						stroke="currentColor"
						stroke-width="2"
					>
						<path d="M12 5v14M5 12h14"></path>
					</svg>
					Add Class
				</button>
				<button
					class="btn btn-primary"
					on:click={logic.generateSchedules}
					disabled={$generatingSchedules}
				>
					<svg
						xmlns="http://www.w3.org/2000/svg"
						width="16"
						height="16"
						viewBox="0 0 24 24"
						fill="none"
						stroke="currentColor"
						stroke-width="2"
					>
						<path d="M3 6l9 6 9-6M3 12l9 6 9-6"></path>
					</svg>
					{$generatingSchedules
						? "Generating..."
						: "Generate Schedules"}
				</button>
			</div>
		</div>
	</header>

	<main class="app-main">
		<div class="restrictions-container">
			<!-- Class Selection -->
			<div class="restriction-box">
				<h2 class="restriction-title">Selected Classes</h2>
				<div class="restriction-items">
					{#each $addedClasses as className, i}
						<button
							class="restriction-button"
							on:click={() => {
								$showClassModals[i] = true;
								$showClassModals = [...$showClassModals];
							}}
							style="background-color: {$colorMap.get(
								className,
							) || '#f5f5f5'}; 
								   color: {$colorMap.get(className) ? '#333' : '#666'};
								   border-color: {$colorMap.get(className)
								? logic.shade(
										$colorMap.get(className) ?? '#646464',
										20,
									)
								: '#e0e0e0'};"
						>
							{className}
						</button>
					{/each}
				</div>
			</div>

			<!-- Professor Restrictions -->
			<div class="restriction-box">
				<h2 class="restriction-title">Professor Restrictions</h2>
				<div class="restriction-items">
					{#each $prohibitedProfessors as prof, i}
						<button
							class="restriction-button professor-restriction"
							on:click={() => {
								$showProfessorModals[i] = true;
								$showProfessorModals = [
									...$showProfessorModals,
								];
							}}
						>
							{prof}
						</button>
					{/each}
				</div>
			</div>

			<!-- Time Restrictions -->
			<div class="restriction-box">
				<h2 class="restriction-title">Time Restrictions</h2>
				<div class="restriction-items">
					{#each $prohibitedTimes as time, i}
						<button
							class="restriction-button time-restriction"
							on:click={() => logic.removeProhibitedTime(i)}
						>
							{logic.formatTimeRestriction(time)}
							<svg
								class="remove-icon"
								xmlns="http://www.w3.org/2000/svg"
								width="14"
								height="14"
								viewBox="0 0 24 24"
								fill="none"
								stroke="currentColor"
								stroke-width="2"
							>
								<line x1="18" y1="6" x2="6" y2="18"></line>
								<line x1="6" y1="6" x2="18" y2="18"></line>
							</svg>
						</button>
					{/each}
				</div>
			</div>
		</div>

		<!-- Display error message if present -->
		{#if $error}
			<div class="error-message" role="alert">
				<svg
					xmlns="http://www.w3.org/2000/svg"
					width="20"
					height="20"
					viewBox="0 0 24 24"
					fill="none"
					stroke="currentColor"
					stroke-width="2"
				>
					<circle cx="12" cy="12" r="10"></circle>
					<line x1="12" y1="8" x2="12" y2="12"></line>
					<line x1="12" y1="16" x2="12.01" y2="16"></line>
				</svg>
				{$error}
			</div>
		{/if}

		<!-- Loading, empty state, or schedules -->
		{#if $generatingSchedules}
			<div class="loading-container">
				<div class="spinner"></div>
				<p>Generating schedules, please wait...</p>
			</div>
		{:else if sortedSchedules.length === 0}
			<div class="empty-state">
				<div class="empty-state-icon">
					<svg
						xmlns="http://www.w3.org/2000/svg"
						width="64"
						height="64"
						viewBox="0 0 24 24"
						fill="none"
						stroke="currentColor"
						stroke-width="1"
					>
						<rect x="3" y="4" width="18" height="18" rx="2" ry="2"
						></rect>
						<line x1="16" y1="2" x2="16" y2="6"></line>
						<line x1="8" y1="2" x2="8" y2="6"></line>
						<line x1="3" y1="10" x2="21" y2="10"></line>
					</svg>
				</div>
				<h2>No schedules generated yet</h2>
				<p>
					Add some classes and click "Generate Schedules" to get
					started
				</p>
			</div>
		{:else}
			<div class="schedules-header">
				<h2>Generated Schedules</h2>
				<div class="schedules-count">
					<span class="badge">{$generatedSchedules.length}</span> possible
					schedules found
				</div>
			</div>
			<div class="schedules-container">
				{#each sortedSchedules as schedule, i (i)}
					<ScheduleView
						scheduleData={schedule}
						colorMap={$colorMap}
						scheduleIndex={i}
					/>
				{/each}
			</div>
		{/if}
	</main>
</div>

<!-- Modals -->
{#if $showTimeSelectionModal}
	<TimeSelectionModal
		prohibitedTimes={$prohibitedTimes}
		showTimeSelectionModal={$showTimeSelectionModal}
		on:close={() => ($showTimeSelectionModal = false)}
		on:add={(event) => {
			// Use the new function to prevent duplicates
			const result = addTimeRestriction(event.detail, $prohibitedTimes);
			$prohibitedTimes = result.map((item) =>
				item instanceof Map ? item : new Map(Object.entries(item)),
			);
			$showTimeSelectionModal = false;
		}}
		on:addMultiple={(event) => {
			// Use the new function to prevent duplicates for multiple restrictions
			$prohibitedTimes = addMultipleTimeRestrictions(
				event.detail,
				$prohibitedTimes,
			).map((item) =>
				item instanceof Map ? item : new Map(Object.entries(item)),
			);
			$showTimeSelectionModal = false;
		}}
	/>
{/if}

{#if $showAddClassModal}
	<AddClassModal
		availableClasses={$availableClasses}
		addedClasses={$addedClasses}
		showAddClassModal={$showAddClassModal}
		colorMap={$colorMap}
		showModals={$showClassModals}
		on:close={() => ($showAddClassModal = false)}
		on:add={(event) => {
			const data = event.detail;
			const className = typeof data === "string" ? data : data.className;

			if (!$addedClasses.includes(className)) {
				// Add the class
				$addedClasses = [...$addedClasses, className];

				// Fetch professor information for this class
				logic.fetchClassDetails(className);

				// Generate a color for the class
				if (!$colorMap.has(className)) {
					$colorMap.set(
						className,
						logic.generateColor($addedClasses.length - 1),
					);
				}

				// Remove from available classes
				$availableClasses = $availableClasses.filter(
					(c) => c !== className,
				);

				// Update showClassModals array
				$showClassModals = new Array($addedClasses.length).fill(false);

				// If event.detail is an object with showModal set to true, show the modal
				if (typeof data === "object" && data.showModal) {
					// Set the modal flag for the newly added class to true
					$showClassModals[$addedClasses.length - 1] = true;
				}

				$showAddClassModal = false;
			}
		}}
	/>
{/if}

{#each $addedClasses as _, i}
	{#if $showClassModals[i]}
		<ClassModal
			className={$addedClasses[i]}
			showModal={$showClassModals[i]}
			prohibitedProfessors={$prohibitedProfessors}
			index={i}
			showProfessorModals={$showProfessorModals}
			closeModal={logic.closeClassModal}
			removeClasses={logic.removeClasses}
			prohibitProf={logic.prohibitProf}
			reAddProfessor={logic.reAddProfessor}
			on:viewProfessor={logic.viewProfessorDetails}
		/>
	{/if}
{/each}

{#each $viewProfessorInfo as profInfo, i}
	{#if profInfo.showModal}
		<ProfessorModal
			profName={profInfo.name}
			addedClasses={$addedClasses}
			showModal={profInfo.showModal}
			index={i}
			closeModal={logic.closeViewProfessorModal}
			reAddProfessor={logic.reAddProfessor}
			isViewOnly={true}
			returnTo={profInfo.returnTo}
		/>
	{/if}
{/each}

{#each $prohibitedProfessors as prof, i}
	{#if $showProfessorModals[i]}
		<ProfessorModal
			profName={prof}
			addedClasses={$addedClasses}
			showModal={$showProfessorModals[i]}
			index={i}
			closeModal={logic.closeProfModal}
			reAddProfessor={logic.reAddProfessor}
		/>
	{/if}
{/each}

<style>
	.app-container {
		display: flex;
		flex-direction: column;
		min-height: 100vh;
		width: 100%; /* Full width */
		margin: 0; /* No margin */
		padding: 0; /* No padding at container level */
		overflow-x: hidden; /* Prevent horizontal scroll */
	}

	.app-header {
		background-color: white;
		padding: var(--space-4) 0;
		box-shadow: var(--shadow-sm);
		border-bottom: 1px solid var(--neutral-200);
		position: sticky;
		top: 0;
		z-index: 10;
		width: 100%; /* Full width */
	}

	.header-content {
		width: 100%;
		max-width: 1400px; /* Content area max width */
		margin: 0 auto;
		padding: 0 var(--space-4); /* Padding on the content */
		display: flex;
		justify-content: space-between;
		align-items: center;
	}

	.app-title {
		font-size: 1.5rem;
		font-weight: 700;
		margin: 0;
		color: var(--primary);
	}

	.header-actions {
		display: flex;
		gap: var(--space-3);
	}

	.app-main {
		padding: var(--space-5) var(--space-4);
		flex: 1;
		width: 100%;
		max-width: 1400px;
		margin: 0 auto;
		overflow-x: hidden; /* Ensure content doesn't overflow */
	}

	/* Restrictions styling */
	.restrictions-container {
		display: grid;
		grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
		gap: var(--space-5);
		margin-bottom: var(--space-6);
	}

	.restriction-box {
		background-color: white;
		border-radius: var(--radius-lg);
		box-shadow: var(--shadow-sm);
		overflow: hidden;
		transition: var(--transition-normal);
	}

	.restriction-box:hover {
		box-shadow: var(--shadow-md);
	}

	.restriction-title {
		padding: var(--space-3) var(--space-4);
		font-size: 1rem;
		font-weight: 600;
		color: var(--neutral-800);
		background-color: var(--neutral-50);
		border-bottom: 1px solid var(--neutral-200);
		margin: 0;
	}

	.restriction-items {
		padding: var(--space-3) var(--space-4);
		display: flex;
		flex-wrap: wrap;
		gap: var(--space-2);
		min-height: 80px;
	}

	.restriction-button {
		border-radius: var(--radius-full);
		font-size: 0.875rem;
		padding: var(--space-1) var(--space-3);
		border: 1px solid var(--neutral-300);
		background-color: var(--neutral-100);
		color: var(--neutral-800);
		display: inline-flex;
		align-items: center;
		gap: var(--space-2);
		cursor: pointer;
		transition: var(--transition-fast);
	}

	.restriction-button:hover {
		transform: translateY(-1px);
		box-shadow: var(--shadow-sm);
	}

	.professor-restriction {
		background-color: var(--neutral-100);
		border-color: var(--neutral-300);
	}

	.time-restriction {
		background-color: var(--primary-light);
		border-color: #f8d7dc;
		padding-right: var(--space-2);
	}

	.remove-icon {
		color: var(--neutral-500);
		margin-left: var(--space-1);
	}

	/* Empty state */
	.empty-state {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		padding: var(--space-7);
		text-align: center;
		color: var(--neutral-600);
		background-color: white;
		border-radius: var(--radius-lg);
		box-shadow: var(--shadow-sm);
	}

	.empty-state-icon {
		color: var(--neutral-400);
		margin-bottom: var(--space-4);
	}

	.empty-state h2 {
		font-size: 1.25rem;
		font-weight: 600;
		margin-bottom: var(--space-2);
		color: var(--neutral-800);
	}

	.empty-state p {
		max-width: 400px;
		color: var(--neutral-600);
	}

	/* Loading spinner */
	.loading-container {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		padding: var(--space-6);
	}

	.spinner {
		width: 40px;
		height: 40px;
		border: 4px solid rgba(226, 24, 51, 0.1);
		border-top: 4px solid var(--primary);
		border-radius: 50%;
		animation: spin 1s linear infinite;
		margin-bottom: var(--space-4);
	}

	@keyframes spin {
		0% {
			transform: rotate(0deg);
		}
		100% {
			transform: rotate(360deg);
		}
	}

	/* Schedules */
	.schedules-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: var(--space-4);
	}

	.schedules-header h2 {
		margin: 0;
		font-size: 1.25rem;
		color: var(--neutral-900);
	}

	.schedules-count {
		color: var(--neutral-600);
		font-size: 0.875rem;
		display: flex;
		align-items: center;
	}

	.badge {
		background-color: var(--primary);
		color: white;
		border-radius: var(--radius-full);
		padding: 2px 8px;
		font-weight: 600;
		margin-right: var(--space-2);
		font-size: 0.75rem;
	}

	.schedules-container {
		display: flex;
		flex-direction: column;
		gap: var(--space-5);
		width: 100%;
		overflow-x: hidden;
	}

	/* Error message */
	.error-message {
		background-color: rgba(244, 67, 54, 0.1);
		border-left: 4px solid var(--error);
		padding: var(--space-3) var(--space-4);
		margin: var(--space-4) 0;
		border-radius: var(--radius-md);
		color: var(--neutral-800);
		display: flex;
		align-items: center;
		gap: var(--space-3);
	}

	/* Empty state indicators */
	.restriction-items:empty::before {
		content: "None added";
		color: var (--neutral-500);
		font-style: italic;
		font-size: 0.875rem;
		width: 100%;
		text-align: center;
		padding: var(--space-3) 0;
	}

	/* Responsive styles */
	@media (max-width: 768px) {
		.header-content {
			flex-direction: column;
			align-items: flex-start;
			gap: var(--space-3);
		}

		.header-actions {
			width: 100%;
			flex-wrap: wrap;
			gap: var(--space-2);
		}

		.btn {
			flex: 1;
			min-width: 0; /* Allow buttons to shrink */
			font-size: 0.85rem; /* Smaller text on mobile */
		}

		.restrictions-container {
			grid-template-columns: 1fr;
		}

		/* Fix potential overflow in the schedule view */
		.schedules-container {
			width: 100%;
			overflow-x: auto; /* Allow horizontal scroll if needed */
		}

		.app-main {
			padding: var(--space-3) var(--space-3);
		}

		/* Make buttons stack vertically on very small screens */
		.header-actions {
			flex-direction: column;
			width: 100%;
		}
	}
</style>
